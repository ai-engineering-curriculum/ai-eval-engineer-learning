# exercise-04: Confidence Sequence for Continuous Monitoring

**Estimated effort:** 3 hours

## Objective

Implement the **`eval_delta` interface** from chapter 05 — an anytime-valid two-sample inference procedure over the scored-row stream — and use it to replace the fixed-`n` t-tests in the exercise-02 drift monitor and the exercise-03 canary gate. Along the way, simulate the peeking-inflation failure mode against a naive t-test so the properties of anytime-valid inference are demonstrated, not just cited.

By the end you will have a small library the rest of the module depends on. When you inherit it into a codebase six months from now, someone will thank you for it.

## Prerequisites

- Chapter 05 of this module.
- Exercise-01's scored-row store, or a simulated stream with the same schema.
- Python 3.11+, `numpy`, `scipy`. Optional but recommended: `confseq` (the Ramdas group's reference implementation for Empirical-Bernstein and betting-style CS).
- Familiarity with mSPRT and confidence sequences at a working level. Read the chapter 05 references before starting; the papers are short and the implementations follow the papers directly.

## Set-up

1. Create `eval/online/stats/` with a small library layout:

   ```
   eval/online/stats/
   ├── __init__.py
   ├── cs_hoeffding.py      # Hoeffding CS for bounded r.v.
   ├── cs_empirical_bernstein.py   # Empirical-Bernstein CS
   ├── cs_betting.py        # betting-style CS (optional)
   ├── msprt.py             # mSPRT for continuous means
   ├── glr.py               # GLR for Bernoulli/rate metrics
   ├── eval_delta.py        # the interface
   ├── multiple_testing.py  # BH correction wrapper
   ├── notebooks/
   │   └── peeking_inflation.ipynb
   └── tests/
       ├── test_cs_coverage.py
       ├── test_msprt_error_rates.py
       └── test_multiple_testing.py
   ```

2. Pull the `confseq` reference implementation into `requirements.txt` if you want a canonical implementation to compare against; own implementations against the papers are also acceptable.

## Requirements

Produce a PR against your working branch that adds:

1. **`cs_hoeffding.py`** — a Hoeffding-style anytime-valid CS on the running mean of a bounded random variable `X ∈ [0, 1]`. Interface: `def hoeffding_cs(values: np.ndarray, alpha: float) -> tuple[float, float]` returning the lower and upper CI on the running mean at time `n = len(values)`. Reference the paper cited in `resources.md`; implement the boundary directly.
2. **`cs_empirical_bernstein.py`** — an Empirical-Bernstein anytime-valid CS on the running mean; adaptively uses the observed variance for tighter intervals on low-variance streams. Same interface as Hoeffding.
3. **`msprt.py`** — a mixture SPRT for a two-sample continuous-mean test. Interface: `def msprt_two_sample(x: np.ndarray, y: np.ndarray, alpha: float, mixing_prior_sigma: float) -> str` returning `"REJECT_NULL"`, `"ACCEPT_NULL"`, or `"CONTINUE"`. Reference the Optimizely Statistical Engine paper (see `resources.md`) or the Deng-Xu-Kohavi treatment for the derivation.
4. **`glr.py`** — a GLR test for a two-sample Bernoulli / rate metric with Chernoff-style boundaries. Same interface shape.
5. **`eval_delta.py`** — the chapter 05 interface exactly:

   ```python
   from dataclasses import dataclass

   @dataclass
   class SeqResult:
       verdict: str
       point_estimate: float
       ci_lower: float
       ci_upper: float
       n_effective: float
       reason: str

   def eval_delta(
       candidate_scores: list[tuple[float, float]],
       reference_scores: list[tuple[float, float]],
       *,
       alpha: float = 0.05,
       null_delta: float = 0.0,
       method: str = "eb_cs",
   ) -> SeqResult:
       ...
   ```

   Supports methods `"eb_cs"`, `"hoeffding_cs"`, `"msprt"`, `"glr"`. The `(score, sample_weight)` pairs flow through as inverse-probability-weighted contributions to `n_effective` and to the running mean.
6. **`multiple_testing.py`** — Benjamini–Hochberg correction over a family of `SeqResult` objects. Interface: `def bh_correct(results: dict[str, SeqResult], family_wide_fdr: float) -> dict[str, bool]` returning per-test reject decisions.
7. **Unit tests**:
   - `test_cs_coverage.py` — simulate 10 000 streams from a fixed distribution, run the CS at every `n`, confirm empirical coverage ≥ `1 - α` at every `n` (this is the anytime-valid property).
   - `test_msprt_error_rates.py` — simulate 10 000 streams under the null (equal means) and confirm empirical Type-I rate ≤ α; simulate 10 000 streams with a specified effect size and confirm Type-II rate ≤ β on average sample size ≤ 80 % of a fixed-`n` t-test with equal power.
   - `test_multiple_testing.py` — simulate a family of 100 tests, 90 % from the null, run BH at FDR = 0.05, confirm empirical FDR ≤ 0.05.
8. **`notebooks/peeking_inflation.ipynb`** — a small demonstration notebook:
   - Simulate 1 000 streams of 10 000 samples each from a null distribution (equal means).
   - Run a naive t-test at every `n` from 100 to 10 000; stop on `p < 0.05`; record the fraction of streams that stopped.
   - Run `eval_delta` with `method="eb_cs"` at every `n`; stop on `REJECT_NULL`; record the fraction.
   - Plot both curves; the naive test's stopping fraction should approach ~30 – 50 % over the observation period; the CS-based stopping fraction should stay ≤ α.

   This notebook is *the* deliverable of the exercise for anyone who has not built the intuition before. Include a plain-language commentary on what the plot shows.
9. **Wire into exercise-02 and exercise-03.** Update `output_drift.py` to call `eval_delta` on the sliding-window scored-row stream instead of the two-sample KS on the mean. Update `canary.sh`'s `gate_lib.py` to call `eval_delta` for the delta-vs-reference test. Confirm both exercises still pass their acceptance criteria; the failure-mode demonstrations from those exercises should now reach decisions faster on real regressions and never on the peeking-null.

## Starter guidance

- **Implement Hoeffding first; it is a five-line boundary.** Empirical-Bernstein adds a variance-tracking step. `betting` is the tightest but the most involved; treat it as optional.
- **The `sample_weight` handling is the load-bearing detail.** With uniform sampling every weight is 1.0 and `n_effective = n`. With biased sampling, `n_effective = (Σ w)² / Σ w²`; the CS width is a function of `n_effective`, not raw `n`. Failing to compute the effective sample size is what makes weighted CIs wrong.
- **The peeking notebook is not optional.** The exercise is teaching the *reason* the machinery exists. Ship the notebook; run it; commit the rendered plots.
- **Compare against `confseq` as a sanity check.** Independently-implemented boundaries should agree with the reference library on the same data to within numerical tolerance. Disagreement is a bug in one of the two; the reference library is more likely to be right.
- **Do not conflate `alpha` and the per-family FDR.** `alpha` in `eval_delta` is the per-test rate; multiple-testing correction on top adjusts the family-wide false-discovery rate. Both are needed; neither replaces the other.
- **Be honest about the width penalty.** Anytime-valid CIs are wider than fixed-`n` CIs. On the peeking notebook, note the trade — the fixed-`n` naive-peek test stops earlier on the null (which is the *bug*) but on a real effect the CS still detects with only modest additional sample size.
- **`INSUFFICIENT_DATA` vs `CONTINUE`.** The gate at exercise-03 returns `INSUFFICIENT_DATA` when the effective sample size is below the pre-registered floor; the CS itself returns `CONTINUE` when the interval still straddles the null. These are different states; distinguish them in the return value.

## Acceptance criteria

You are done when:

- `cs_hoeffding.py` and `cs_empirical_bernstein.py` implement the anytime-valid CI boundaries; the coverage unit test passes for a simulated null distribution over ≥ 10 000 replicates.
- `msprt.py` and `glr.py` control Type-I rate at α and reach Type-II decisions in ≤ 80 % of the fixed-`n` sample size at equal power.
- `eval_delta` correctly propagates `sample_weight` through to `n_effective` and to the running mean.
- The peeking-inflation notebook demonstrates the false-positive rate of the naive t-test approaching ~30 – 50 % over the observation period, and the CS-based procedure staying ≤ α.
- Exercise-02's output-drift monitor and exercise-03's canary gate compile against the new library and pass their acceptance criteria against real or synthesised streams.
- BH correction is applied on top; unit test confirms empirical FDR ≤ target.
- A `README.md` in `eval/online/stats/` explains when to use `eb_cs` (bounded scores), `hoeffding_cs` (binary), `msprt` (fixed-hypothesis two-sample), and `glr` (change-point on rates).

## Stretch goals

- **Betting-style CS.** Implement `cs_betting.py` using the Waudby-Smith and Ramdas construction; compare interval width to Empirical-Bernstein on the same real scored-row streams from exercise-01; report which is tighter under what conditions.
- **Change-point detection.** Extend `glr.py` to return not just `REJECT_NULL` but also the estimated change-point time; wire into the exercise-02 drift monitor as an additional runbook signal ("the drift started ~12h ago").
- **Sequential-mode as a canary option.** Extend `gate_lib.py` in exercise-03 to accept a `--sequential` flag; on the same rehearsal script (regression injected mid-window) confirm the canary reaches `FAIL` earlier than the fixed-window variant.
- **Cluster-robust weighting.** Extend `eval_delta` to accept a `cluster_id` per row (session / tenant) and compute cluster-robust `n_effective`; useful when many scored rows come from a single session and are correlated.
- **Interactive visualisation.** Build a small Streamlit or Panel app that lets an eval engineer paste in a real scored-row stream and inspect the CS width over time, the peeking-null false-positive rate, and the effect size at which the CS closes.
- **Cross-language port.** Port the CS boundaries to TypeScript or Go if the online-loop worker in your surface's language is not Python; confirm parity to numerical tolerance against the Python reference.

## What this exercise does *not* cover

You are not building the A/B interface (exercise-05) or the dashboards (exercise-06). You are shipping the anytime-valid inference library that both would consume, and you are updating exercises 02 and 03 to use it in place of naive fixed-`n` tests.
