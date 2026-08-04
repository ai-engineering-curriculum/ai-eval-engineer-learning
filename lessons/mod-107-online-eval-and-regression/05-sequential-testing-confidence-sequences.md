# Sequential Testing and Confidence Sequences for Continuous Monitoring

## Motivation

Chapter 03's drift monitor and chapter 04's canary gate both read the score stream *continuously*. A drift monitor watches the last 24 hours as the window slides every minute. A canary gate re-computes the delta each time a new scored row lands. These are not one-shot tests — they are re-tests. The naive move is to run a standard fixed-`n` two-sample t-test every time, alert when `p < 0.05`, and move on.

That move is wrong. Continuously re-evaluating a fixed-`n` p-value against traffic that keeps arriving is what the statistics literature calls **peeking**. Under continuous monitoring, a nominal α = 0.05 test on a stream of accumulating data has a true false-positive rate that approaches 1 as the observation period lengthens — in the long run, you will *always* eventually see a `p < 0.05` window under the null. This is the mathematical reason "run the t-test again in 5 minutes" is a pager-generator, not a monitor.

The fix is a family of statistical procedures collectively called **sequential testing** or **anytime-valid inference**: procedures whose false-positive rate is controlled *for the entire observation period, at any stopping time you like*, not just at a pre-specified `n`. Two members of the family cover 90 % of what an online-eval program needs.

- **Sequential probability ratio tests (mSPRT / GLR)** — for a fixed hypothesis (`H_0: μ_treatment = μ_control`), continuously compute a likelihood ratio and stop as soon as it crosses a threshold.
- **Confidence sequences** — a time-uniform confidence interval that shrinks as more data arrives, and whose coverage guarantee holds at *every* time step simultaneously. When zero (or the pre-registered floor) is outside the interval, you have your decision.

This chapter walks the machinery, states when each fits, and shows the interface the online loop uses.

## Core concepts

### Why peeking is fatal, quantified

The nominal Type-I rate of a t-test is 5 %. Under continuous monitoring, an experimenter who stops as soon as `p < 0.05` for the first time can achieve a true Type-I rate close to 100 % if the observation period is long enough — the "law of iterated logarithm" bound on the running sample mean means the running z-statistic drifts above any fixed threshold infinitely often under the null.

Empirical anchors are less dramatic but still bad. Simulation studies routinely find that stopping-first-crossing on a naive t-test after up to `N` observations yields effective false-positive rates of ~20 % for `N` = 1 000 and ~30 % for `N` = 10 000, against a nominal 5 %. Details in the Optimizely / Statsig / Deng-Xu-Kohavi papers cited in `resources.md`; the takeaway does not depend on the exact number: **repeated peeking inflates the false-positive rate, unbounded in `N`**.

Two consequences the online loop must respect:

- **A fixed-`n` t-test on a sliding-window aggregate is *not* a valid drift alert.** It works exactly as advertised if you commit up front to one look at one specific `n` and never look again — which is not the online loop.
- **A p-value from such a test is not comparable to the pre-registered α.** Reporting `p = 0.03` on a peeked stream and comparing it to α = 0.05 is comparing two different things.

### The alternative: control α *across* the whole stream

Two design choices, both mathematically sound.

- **Alpha-spending.** Divide the total α across pre-registered "looks" — 10 looks at α = 0.005 each, say. Simple; scales badly (`O(1/n)` per look, so a monitor that runs indefinitely has zero α per look in the limit); requires committing to the look schedule up front.
- **Anytime-valid inference.** The procedure guarantees α *for any stopping rule*. You can look every millisecond; you can look based on what the previous look told you; the guarantee holds. This is what mSPRT, GLR, and confidence sequences deliver.

Anytime-valid is the correct default for the online loop. Alpha-spending is the correct default for the A/B experiment (chapter 06) — an experiment has a finite horizon and a pre-registered look schedule; a monitor does not.

### mSPRT (mixture sequential probability ratio test)

The **mixture sequential probability ratio test** — the workhorse of continuous experimentation at Optimizely, LinkedIn, and Microsoft — is a Wald SPRT with an integrated mixture prior over the effect size. For a two-sample test on means:

- At each new sample, compute the likelihood ratio for `H_1: μ_treatment - μ_control ≠ 0` against `H_0: μ_treatment - μ_control = 0`, integrating over a mixing distribution on the alternative (Gaussian mixture is the most common).
- Reject `H_0` when the ratio crosses `1/α`; accept `H_0` when it crosses `β/(1-α)`; else continue sampling.

Properties:

- The false-positive rate is controlled at α at any stopping time.
- The false-negative rate is controlled at β at any stopping time.
- Expected sample size to a decision is typically 50 – 80 % of a fixed-`n` test with the same power.
- The mixing distribution's width is a knob: wider = more conservative but faster on large effects; narrower = tighter power at the mixing centre.

mSPRT is the natural fit for the canary gate (chapter 04). The gate asks a binary question — "is the candidate different from the reference?" — and can stop as soon as evidence accumulates.

### GLR (generalized likelihood ratio)

The **generalized likelihood ratio** test replaces the fixed alternative in the SPRT with a maximum-likelihood alternative. Instead of integrating over a prior, GLR at each step plugs in the MLE of the effect size and compares:

`Λ_n = sup_{θ ∈ Θ_1} L(θ; data) / L(θ_0; data)`

Combined with a Chernoff-style boundary or a mixture-boundary, GLR is anytime-valid. It generalises to non-Gaussian data more readily than mSPRT (Bernoulli / Poisson / exponential-family metrics) and is the modern default for change-point detection.

Use GLR when the metric is a rate (refusal rate, tool-error rate, safety-block rate) or a count; use mSPRT for continuous scores (rubric scores, latencies). Both live in the same conceptual family; do not agonise over the choice — pick per metric shape and use one consistently.

### Confidence sequences

A **confidence sequence** is a sequence of confidence intervals `[L_n, U_n]`, indexed by `n`, such that `P(∀ n : θ ∈ [L_n, U_n]) ≥ 1 - α`. The coverage guarantee holds simultaneously for *every* `n` — you can stop looking whenever you like and the interval is valid.

The modern construction (Howard, Ramdas, McAuliffe, and Sekhon; see `resources.md`) uses martingale-based boundaries derived from the law of the iterated logarithm. Practical properties:

- The interval only ever *shrinks* as `n` grows; it never widens.
- The interval is wider than a fixed-`n` CI at the same `n` — you pay for the anytime-valid guarantee — but *the payment is small* (roughly a factor of 2 in width for practical `α`).
- Any moment you can say "the interval excludes the pre-registered floor / does not include zero," you have your decision, and the false-positive rate is still α.
- Extensions cover bounded random variables (Hoeffding-style CS), unbounded (Empirical-Bernstein CS), quantiles, and paired data.

For the online loop, the recommended default is a **bounded-variable confidence sequence on the metric mean**, per cohort, per metric. The canary gate (chapter 04) declares `FAIL` iff the upper bound on the delta-vs-reference is below `-|Δ|`; the drift monitor (chapter 03) declares `PAGE` iff the CS excludes the reference-window mean. This is one code path serving both use cases.

### Betting-style confidence sequences

A newer family (Waudby-Smith and Ramdas, 2024) constructs confidence sequences by casting the test as a **betting** problem — a hypothetical better wagers against the null; wealth grows when the alternative is true; wealth crossing a threshold rejects the null. Betting-CS often produce tighter intervals than the Hoeffding-based classical construction, especially for bounded random variables like rubric scores in `[0, 1]`.

The `confseq` Python library (Ramdas group) and the `betting_cs` module in `admiraltymath` implement the main constructions. The paper's key property: `EB_CS` (Empirical Bernstein) confidence sequences adaptively use the observed variance and give near-fixed-`n` width on high-variance metrics.

Practically: use `EB_CS` on rubric scores (bounded `[0, 1]`, unknown variance); use a Hoeffding CS on binary metrics (refusal / no-refusal) because the variance is `p(1-p)` bounded by `0.25` and the Hoeffding is essentially tight; use GLR when you need the *change-point* time as an output, not just "there is a difference".

### Interface: what the online loop calls

Wrap the confidence sequence behind an interface the monitor and the gate both consume.

```python
from typing import Optional
from dataclasses import dataclass

@dataclass
class SeqResult:
    verdict: str            # "REJECT_NULL" | "ACCEPT_NULL" | "CONTINUE"
    point_estimate: float
    ci_lower: float
    ci_upper: float
    n_effective: float
    reason: str

def eval_delta(
    candidate_scores: list[tuple[float, float]],  # (score, sample_weight) pairs
    reference_scores: list[tuple[float, float]],
    *,
    alpha: float = 0.05,
    null_delta: float = 0.0,
    method: str = "eb_cs",
) -> SeqResult:
    """
    Two-sample anytime-valid inference on the mean delta
    (candidate - reference). Returns a CONTINUE verdict when
    the CS still straddles the null_delta; REJECT_NULL when
    the CS excludes it in either direction.
    """
    ...
```

Downstream code:

```python
result = eval_delta(
    candidate_scores=window_scores(canary_cohort, "faithfulness"),
    reference_scores=window_scores(stable_cohort, "faithfulness"),
    alpha=0.01,
    null_delta=-0.02,   # the pre-registered floor delta
)
if result.verdict == "REJECT_NULL" and result.point_estimate < -0.02:
    gate_fail("faithfulness", result)
elif result.verdict == "CONTINUE":
    gate_insufficient("faithfulness", result)
else:
    gate_pass("faithfulness", result)
```

Two implementation notes:

- The function takes `(score, sample_weight)` pairs — chapter 02's inverse-probability weighting flows through the CS. Missing sample weights collapses to uniform.
- `alpha` here is the *per-metric-per-cohort* rate. Multiple-testing correction across metrics × cohorts is applied *outside* — a Benjamini–Hochberg pass on the family of tests before deciding to page.

### Multiple testing across metrics and cohorts

The online loop runs the confidence sequence *many times* — one per metric per cohort per window. Twenty metrics × twenty cohorts × one alert per minute is 8 000 tests per minute; at α = 0.05, that is 400 spurious pages per minute even with anytime-valid inference *per test*.

Two moves are the current defaults.

- **Benjamini–Hochberg over the current test family.** Compute the anytime-valid `p`-value equivalent for every metric-cohort pair; apply BH at the family-wide false-discovery rate you can absorb (usually 5 – 10 %); page only on tests that clear BH. Chapter 03 introduced this at the drift-monitor scale; it applies to the canary-stage gate identically.
- **Hierarchical group control.** Group tests by cohort family (locale, tenant tier, region); apply BH within group and Bonferroni across groups. Prevents a locale-specific storm from swamping the aggregate correction.

Anytime-valid inference controls Type-I rate per test; multiple-testing correction controls it across the family. Both matter; neither replaces the other.

### When *not* to use a confidence sequence

Two situations where the simpler tool is right:

- **Pre-registered A/B experiment with a fixed horizon.** The A/B platform (chapter 06) already has a design with a pre-specified sample size, look schedule, and stopping rule. Use its native statistics (usually alpha-spending or a group-sequential design); do not replace with a CS unless you own the platform.
- **Aggregate weekly / monthly report.** A single retrospective look at the last week's aggregate does not need anytime-valid inference — a fixed-`n` CI on the point estimate is correct because the "look" is a single event, not a stream of decisions.

Confidence sequences are the default for **anything you look at more than once during the observation period**. Fixed-`n` is the default for a single look.

### The α-budget and the runbook

Every alert costs the on-call's attention. Anytime-valid inference guarantees the *statistical* false-positive rate; the *behavioural* false-positive rate (alerts that fire on the tail of a benign distribution) is a function of thresholds, cool-downs, and severity gates. A rough budget:

- < 1 page / on-call shift for a healthy program.
- < 5 alerts / week that require action.
- < 20 alerts / week that require inspection.

If the loop exceeds those, the fix is not "lower α further" (that inflates false negatives). The fix is:

- Raise the effect-size threshold — a `null_delta = 0` gate fires on any imperceptible difference; a `null_delta = -0.02` gate only fires on a 2-point drop.
- Add per-metric cool-downs (chapter 03).
- Route warn-level alerts to a dashboard, not the pager.

### Vendor coverage

- **Statsig** and **Eppo** publish and support mSPRT-style continuous testing as the default experiment engine.
- **Optimizely** publishes the original Statistical Engine paper defining the mSPRT bounds used across the industry.
- **`confseq` (Python, Ramdas group)** — the reference implementation for betting-style and Empirical-Bernstein confidence sequences.
- **`Sequential` (R, Kulldorff & Silva)** — for maximum sequential probability ratio test on rates / counts.
- **Alibi Detect (Python)** — a subset of drift detectors have sequential variants; check the module before assuming.

## Summary

- **Peeking on a fixed-`n` test inflates the false-positive rate without bound.** Continuously monitoring a naive `p < 0.05` t-test is a pager generator, not a monitor.
- Two design responses: **alpha-spending** (works for pre-registered look schedules; A/B platform's default) and **anytime-valid inference** (works for arbitrary stopping rules; the online loop's default).
- Two anytime-valid families: **mSPRT / GLR** (a hypothesis test that stops as soon as evidence accumulates) and **confidence sequences** (a time-uniform CI that only shrinks).
- Recommended default: **empirical-Bernstein confidence sequences** on bounded rubric scores; **Hoeffding CS** on binary metrics; **GLR** when you need the change-point time.
- **One interface** — `eval_delta(candidate, reference, alpha, null_delta)` — serves both the canary gate (chapter 04) and the drift monitor (chapter 03).
- **Multiple-testing correction** across metrics × cohorts is layered on top; Benjamini–Hochberg is the default at production scale.
- **Not everything needs a CS**: pre-registered A/B experiments use their platform's design; weekly retrospective reports use a fixed-`n` CI.
- The α-budget is a *behavioural* number; if the loop over-fires, raise the effect-size threshold and add cool-downs before lowering α.

Chapter 06 covers the interface with the A/B platform — the pre-registration contract, the CUPED variance-reduction covariate, and where model-evaluation-engineer / senior-ml-engineer take over the deep experimental methodology.
