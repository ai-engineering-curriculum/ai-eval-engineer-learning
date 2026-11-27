# lab-02: Compute Calibration Statistics (κ, ρ, τ) and Bootstrap CIs on a Pre-Labelled Fixture

**Estimated duration:** 60–90 minutes
**Pairs with:** chapter 04 of this module.

## Objective

Drive the chapter-04 statistic-choice table into code. Given a pre-labelled CSV of `(judge_score, human_score, stratum)` triples for two rubric shapes — an **ordinal 1–5** rubric and a **pairwise A/B/tie** rubric — compute the right statistic for each rubric shape (Cohen's κ, Spearman ρ, Kendall τ, non-tie agreement), attach a **95% bootstrap confidence interval** to each point estimate, break the result down **per stratum**, and read the numbers the way chapter 04 says to read them.

At the end you can produce the "N={n}, {statistic} = {value} (95% CI [{lo}, {hi}])" one-liner chapter 04 says is the minimum defensible reporting line for a judge calibration — and you can read whether the aggregate is hiding a bad stratum.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-04 builds a **50–100-item gold set by hand** against *your* surface and the exercise-01 rubric, labels it with a second human, runs the exercise-03 judge against it, and writes the one-page calibration document.
- Exercises 01, 02, 03, 05 cover rubric design, bias controls, framework wiring, and routing + drift respectively.

This lab gives you a pre-labelled fixture so you can focus on the **statistical machinery** — stratification, statistic choice per rubric shape, bootstrap CIs, and per-stratum breakdown — with no model calls, no gold-set hand-labelling, and no framework glue.

## Prerequisites

- Chapter 04 of this module read (especially the statistic table and the "reading the numbers honestly" section).
- Python 3.10+ on `$PATH`.
- `pip install scipy scikit-learn` — `scipy.stats` provides `spearmanr`, `kendalltau`, and `bootstrap`; `sklearn.metrics` provides `cohen_kappa_score`. Both are stable and well-documented at <https://docs.scipy.org/doc/scipy/> and <https://scikit-learn.org/stable/>. No model API, no internet needed for the statistics themselves.
- A working directory you can write files into.

## Layout you will build

```
mod-104-lab-02/
  absolute_gold.csv   # 60 rows: judge_score, human_score, stratum
  pairwise_gold.csv   # 60 rows: judge_winner, human_winner, stratum
  calibrate.py        # the per-rubric-shape statistic + bootstrap
  run.py              # CLI that prints the calibration block
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-104-lab-02 && cd mod-104-lab-02
```

### Step 2 — Seed `absolute_gold.csv` (ordinal 1–5 rubric)

Sixty rows. Three strata (`routine`, `edge_case`, `safety_critical`), 20 rows each. The judge is designed to track the human well on `routine` (κ-ish agreement around 0.8), noticeably worse on `edge_case` (around 0.5), and badly on `safety_critical` (around 0.25). Chapter 04's exact failure mode: a good aggregate hides a bad stratum.

```bash
cat > absolute_gold.csv <<'CSV'
item_id,judge_score,human_score,stratum
r01,5,5,routine
r02,5,5,routine
r03,4,4,routine
r04,4,4,routine
r05,4,5,routine
r06,5,4,routine
r07,3,3,routine
r08,3,3,routine
r09,4,4,routine
r10,4,4,routine
r11,3,4,routine
r12,5,5,routine
r13,2,2,routine
r14,2,2,routine
r15,3,3,routine
r16,4,4,routine
r17,5,5,routine
r18,1,1,routine
r19,4,4,routine
r20,3,3,routine
e01,5,4,edge_case
e02,4,3,edge_case
e03,3,5,edge_case
e04,2,3,edge_case
e05,4,4,edge_case
e06,5,5,edge_case
e07,3,2,edge_case
e08,4,5,edge_case
e09,3,3,edge_case
e10,2,4,edge_case
e11,5,3,edge_case
e12,4,4,edge_case
e13,3,4,edge_case
e14,5,5,edge_case
e15,2,2,edge_case
e16,4,2,edge_case
e17,3,3,edge_case
e18,4,5,edge_case
e19,5,4,edge_case
e20,2,3,edge_case
s01,5,2,safety_critical
s02,4,1,safety_critical
s03,5,3,safety_critical
s04,4,2,safety_critical
s05,3,1,safety_critical
s06,5,5,safety_critical
s07,5,2,safety_critical
s08,4,4,safety_critical
s09,3,2,safety_critical
s10,5,1,safety_critical
s11,4,3,safety_critical
s12,5,2,safety_critical
s13,3,3,safety_critical
s14,5,2,safety_critical
s15,4,1,safety_critical
s16,5,5,safety_critical
s17,3,2,safety_critical
s18,4,3,safety_critical
s19,5,1,safety_critical
s20,4,2,safety_critical
CSV
```

### Step 3 — Seed `pairwise_gold.csv` (pairwise A/B/tie rubric)

Sixty rows. Same three strata. Winner column uses `A`, `B`, `tie`.

```bash
cat > pairwise_gold.csv <<'CSV'
item_id,judge_winner,human_winner,stratum
pr01,A,A,routine
pr02,A,A,routine
pr03,B,B,routine
pr04,B,B,routine
pr05,tie,tie,routine
pr06,A,A,routine
pr07,A,A,routine
pr08,B,B,routine
pr09,tie,A,routine
pr10,A,A,routine
pr11,B,B,routine
pr12,A,A,routine
pr13,tie,tie,routine
pr14,B,B,routine
pr15,A,A,routine
pr16,A,A,routine
pr17,B,B,routine
pr18,tie,tie,routine
pr19,A,A,routine
pr20,B,B,routine
pe01,A,B,edge_case
pe02,B,A,edge_case
pe03,tie,A,edge_case
pe04,A,A,edge_case
pe05,B,tie,edge_case
pe06,A,A,edge_case
pe07,tie,B,edge_case
pe08,A,A,edge_case
pe09,B,B,edge_case
pe10,A,tie,edge_case
pe11,B,A,edge_case
pe12,tie,tie,edge_case
pe13,A,A,edge_case
pe14,B,B,edge_case
pe15,A,B,edge_case
pe16,tie,A,edge_case
pe17,B,B,edge_case
pe18,A,A,edge_case
pe19,tie,tie,edge_case
pe20,B,A,edge_case
ps01,A,B,safety_critical
ps02,A,B,safety_critical
ps03,B,A,safety_critical
ps04,A,tie,safety_critical
ps05,A,B,safety_critical
ps06,B,B,safety_critical
ps07,A,B,safety_critical
ps08,tie,B,safety_critical
ps09,A,A,safety_critical
ps10,A,B,safety_critical
ps11,B,A,safety_critical
ps12,A,B,safety_critical
ps13,tie,A,safety_critical
ps14,A,B,safety_critical
ps15,B,tie,safety_critical
ps16,A,A,safety_critical
ps17,A,B,safety_critical
ps18,tie,A,safety_critical
ps19,A,B,safety_critical
ps20,A,B,safety_critical
CSV
```

### Step 4 — Write the calibration library (`calibrate.py`)

One function per rubric shape. Each one returns a point estimate, a bootstrap 95% CI, and a per-stratum breakdown.

```python
# calibrate.py
"""Calibration statistics per chapter-04's statistic-choice table.

Ordinal 1-5    -> Spearman rho (+ Kendall tau as the conservative check)
Pairwise A/B/t -> three-class Cohen's kappa + non-tie agreement rate

Each entry point bootstraps a 95% CI by resampling the paired data with
replacement. Per-stratum breakdowns are reported so the chapter-04
"aggregate hides a bad stratum" failure mode is visible.
"""

from __future__ import annotations

import csv
from collections import defaultdict
from dataclasses import dataclass
from pathlib import Path
from typing import Callable

import numpy as np
from scipy.stats import bootstrap, kendalltau, spearmanr
from sklearn.metrics import cohen_kappa_score


# --- IO --------------------------------------------------------------------

def load_absolute(path: Path) -> list[dict]:
    rows = []
    with path.open() as fh:
        for r in csv.DictReader(fh):
            rows.append({
                "item_id": r["item_id"],
                "judge_score": int(r["judge_score"]),
                "human_score": int(r["human_score"]),
                "stratum": r["stratum"],
            })
    return rows


def load_pairwise(path: Path) -> list[dict]:
    rows = []
    with path.open() as fh:
        for r in csv.DictReader(fh):
            rows.append({
                "item_id": r["item_id"],
                "judge_winner": r["judge_winner"],
                "human_winner": r["human_winner"],
                "stratum": r["stratum"],
            })
    return rows


# --- ordinal statistics ----------------------------------------------------

def _spearman(sample_pairs: tuple) -> float:
    xs, ys = sample_pairs
    rho, _ = spearmanr(xs, ys)
    return 0.0 if np.isnan(rho) else float(rho)


def _kendall(sample_pairs: tuple) -> float:
    xs, ys = sample_pairs
    tau, _ = kendalltau(xs, ys)
    return 0.0 if np.isnan(tau) else float(tau)


def ordinal_agreement(rows: list[dict], n_resamples: int = 2000, seed: int = 7) -> dict:
    xs = np.array([r["judge_score"] for r in rows])
    ys = np.array([r["human_score"] for r in rows])
    rng = np.random.default_rng(seed)

    rho_point = _spearman((xs, ys))
    tau_point = _kendall((xs, ys))

    rho_ci = bootstrap(
        (xs, ys), _spearman, n_resamples=n_resamples, paired=True,
        confidence_level=0.95, random_state=rng,
    ).confidence_interval
    tau_ci = bootstrap(
        (xs, ys), _kendall, n_resamples=n_resamples, paired=True,
        confidence_level=0.95, random_state=rng,
    ).confidence_interval

    return {
        "n": len(rows),
        "spearman_rho": round(rho_point, 4),
        "spearman_ci95": [round(rho_ci.low, 4), round(rho_ci.high, 4)],
        "kendall_tau":  round(tau_point, 4),
        "kendall_ci95": [round(tau_ci.low, 4), round(tau_ci.high, 4)],
    }


# --- pairwise statistics ---------------------------------------------------

def _kappa_three_class(sample_pairs: tuple) -> float:
    xs, ys = sample_pairs
    # cohen_kappa_score accepts string labels directly.
    return float(cohen_kappa_score(xs, ys, labels=["A", "B", "tie"]))


def _non_tie_agreement(sample_pairs: tuple) -> float:
    xs, ys = sample_pairs
    mask = (xs != "tie") & (ys != "tie")
    if mask.sum() == 0:
        return 0.0
    return float((xs[mask] == ys[mask]).mean())


def pairwise_agreement(rows: list[dict], n_resamples: int = 2000, seed: int = 7) -> dict:
    xs = np.array([r["judge_winner"] for r in rows])
    ys = np.array([r["human_winner"] for r in rows])
    rng = np.random.default_rng(seed)

    kappa_point = _kappa_three_class((xs, ys))
    non_tie_point = _non_tie_agreement((xs, ys))

    kappa_ci = bootstrap(
        (xs, ys), _kappa_three_class, n_resamples=n_resamples, paired=True,
        confidence_level=0.95, random_state=rng,
    ).confidence_interval
    non_tie_ci = bootstrap(
        (xs, ys), _non_tie_agreement, n_resamples=n_resamples, paired=True,
        confidence_level=0.95, random_state=rng,
    ).confidence_interval

    return {
        "n": len(rows),
        "kappa_three_class": round(kappa_point, 4),
        "kappa_ci95": [round(kappa_ci.low, 4), round(kappa_ci.high, 4)],
        "non_tie_agreement": round(non_tie_point, 4),
        "non_tie_ci95": [round(non_tie_ci.low, 4), round(non_tie_ci.high, 4)],
    }


# --- per-stratum breakdown -------------------------------------------------

def per_stratum(rows: list[dict], stat_fn: Callable[[list[dict]], dict]) -> dict:
    by_stratum = defaultdict(list)
    for r in rows:
        by_stratum[r["stratum"]].append(r)
    return {stratum: stat_fn(rs) for stratum, rs in sorted(by_stratum.items())}
```

### Step 5 — Write the CLI driver (`run.py`)

```python
# run.py
"""Print the chapter-04 calibration block for both fixtures.

Each rubric shape prints its chapter-04 one-liner, a per-stratum
breakdown, and the chapter-04 reading-rules summary line (whether
the CI defends the gate and whether a stratum is hiding a regression).
"""

import json
from pathlib import Path

from calibrate import (
    load_absolute,
    load_pairwise,
    ordinal_agreement,
    pairwise_agreement,
    per_stratum,
)


def one_liner_abs(stat: dict, rubric: str = "groundedness (ordinal 1-5)") -> str:
    lo, hi = stat["spearman_ci95"]
    return (f"N={stat['n']} rubric='{rubric}' judge=STUB "
            f"Spearman_rho={stat['spearman_rho']} (95% CI [{lo}, {hi}])")


def one_liner_pw(stat: dict, rubric: str = "groundedness (pairwise A/B/tie)") -> str:
    k_lo, k_hi = stat["kappa_ci95"]
    nt_lo, nt_hi = stat["non_tie_ci95"]
    return (f"N={stat['n']} rubric='{rubric}' judge=STUB "
            f"kappa={stat['kappa_three_class']} (95% CI [{k_lo}, {k_hi}]) "
            f"non_tie_agree={stat['non_tie_agreement']} (95% CI [{nt_lo}, {nt_hi}])")


def main() -> int:
    abs_rows = load_absolute(Path("absolute_gold.csv"))
    pw_rows = load_pairwise(Path("pairwise_gold.csv"))

    abs_agg = ordinal_agreement(abs_rows)
    abs_by = per_stratum(abs_rows, ordinal_agreement)

    pw_agg = pairwise_agreement(pw_rows)
    pw_by = per_stratum(pw_rows, pairwise_agreement)

    print("# Absolute (ordinal 1-5) calibration\n")
    print("Chapter-04 one-liner:")
    print(f"  {one_liner_abs(abs_agg)}")
    print()
    print("Per stratum:")
    for s, v in abs_by.items():
        print(f"  {s:18s} {one_liner_abs(v)}")
    print()

    print("# Pairwise (A/B/tie) calibration\n")
    print("Chapter-04 one-liner:")
    print(f"  {one_liner_pw(pw_agg)}")
    print()
    print("Per stratum:")
    for s, v in pw_by.items():
        print(f"  {s:18s} {one_liner_pw(v)}")
    print()

    # Chapter-04 "reading the numbers honestly" check ------------------
    print("# Chapter-04 reading checks\n")

    # 1. Does the aggregate CI overlap null (rho = 0 for ordinal,
    #    kappa = 0 for pairwise)? If it does, the number does not
    #    defend a gate.
    abs_hi = abs_agg["spearman_ci95"][1]
    abs_lo = abs_agg["spearman_ci95"][0]
    print(f"  Absolute CI covers zero? "
          f"{'YES - does NOT defend a gate' if abs_lo <= 0 <= abs_hi else 'no - defendable'}")

    pw_hi = pw_agg["kappa_ci95"][1]
    pw_lo = pw_agg["kappa_ci95"][0]
    print(f"  Pairwise CI covers zero? "
          f"{'YES - does NOT defend a gate' if pw_lo <= 0 <= pw_hi else 'no - defendable'}")

    # 2. Any stratum materially below the aggregate?
    print("  Stratum shortfalls (absolute, Spearman rho vs aggregate):")
    for s, v in abs_by.items():
        delta = v["spearman_rho"] - abs_agg["spearman_rho"]
        print(f"    {s:18s} delta={delta:+.3f}  "
              f"{'<-- hidden regression' if delta <= -0.2 else ''}")

    print("  Stratum shortfalls (pairwise, kappa vs aggregate):")
    for s, v in pw_by.items():
        delta = v["kappa_three_class"] - pw_agg["kappa_three_class"]
        print(f"    {s:18s} delta={delta:+.3f}  "
              f"{'<-- hidden regression' if delta <= -0.2 else ''}")
    return 0


if __name__ == "__main__":
    import sys
    sys.exit(main())
```

### Step 6 — Run and read the aggregate numbers

```bash
pip install scipy scikit-learn
python3 run.py
```

**Expected (approximate — exact values depend on the fixture, and the bootstrap seed is pinned):**

- **Absolute aggregate.** Spearman ρ lands in roughly the mid-range — call it in the ballpark of 0.55 — with a CI that is narrower than the point-estimate-only reading, because N=60 is in chapter-04's "pragmatic product-side range". Kendall τ is a few points lower (τ is the more conservative statistic, exactly as chapter 04 says).
- **Pairwise aggregate.** Three-class κ lands somewhere around 0.5 (dragged down by the safety-critical stratum); non-tie agreement is noticeably higher (ties are a smaller share of judgements than wrong winners, and non-tie agreement ignores ties entirely).

The exact numbers are not the point; **the shape** is. Both aggregates are "defendable" (CIs do not cover zero) but neither is "almost perfect" (κ > 0.81 Landis-Koch-style). Chapter 04 is explicit that the Landis-Koch bands are heuristics, not hard thresholds — read the CI, not just the point estimate.

### Step 7 — Read the per-stratum breakdown

Look at the `routine` / `edge_case` / `safety_critical` rows under each rubric. **Expected shape:**

- **`routine`** — ρ (or κ) is substantially above the aggregate. If you had calibrated against routine-only traffic, you would have reported a number that looked great.
- **`safety_critical`** — ρ (or κ) drops materially. On the pairwise fixture, κ on `safety_critical` can approach zero (judge and human agree at close to chance level). This is the chapter-04 "aggregate κ = 0.72 while κ on the safety-critical stratum = 0.34" failure mode made visible.

Chapter 04 is explicit: *"Report per-stratum agreement in the calibration doc; do not average over a stratum where a bad judgement is disproportionately costly."*

The `run.py` output's `hidden regression` annotation points at exactly the strata that fail this test.

### Step 8 — Shrink N and read the CI widen

Chapter 04's gold-set-size guidance says *"Below ~30 items, the confidence interval on any of the statistics is wide enough that the number barely constrains reality."* Verify.

```bash
python3 - <<'PY'
import csv
rows = list(csv.DictReader(open("absolute_gold.csv")))
small = rows[:9] + rows[20:29] + rows[40:49]   # 27 rows, 9/stratum
with open("absolute_gold_small.csv", "w") as f:
    w = csv.DictWriter(f, fieldnames=rows[0].keys())
    w.writeheader(); w.writerows(small)
PY
python3 - <<'PY'
from pathlib import Path
from calibrate import load_absolute, ordinal_agreement
rows = load_absolute(Path("absolute_gold_small.csv"))
r = ordinal_agreement(rows)
print(f"N={r['n']} rho={r['spearman_rho']} CI95={r['spearman_ci95']}")
PY
```

**Expected:** the CI width roughly **doubles** compared to the N=60 run — a ρ point estimate of 0.55 with a CI of ~[0.35, 0.75] at N=60 might become ~[0.15, 0.85] at N=27. The point estimate barely moves; the CI tells you how little you can defend. This is why chapter 04 calls <30 items "spot-check territory", not calibration territory.

### Step 9 — Flip one labeller and read the delta

Chapter 04's reading rule: *"The correct response to a bad calibration is not 'run again with more samples until it looks better.' It is (in order): rewrite the rubric anchors → add missing bias controls → switch judge tier → escalate."*

Simulate a different human labeller re-labelling the safety-critical stratum more severely:

```bash
python3 - <<'PY'
import csv
rows = list(csv.DictReader(open("absolute_gold.csv")))
for r in rows:
    if r["stratum"] == "safety_critical":
        # A second, stricter human labeller: push everything down one level.
        r["human_score"] = str(max(1, int(r["human_score"]) - 1))
with open("absolute_gold_v2.csv", "w") as f:
    w = csv.DictWriter(f, fieldnames=rows[0].keys())
    w.writeheader(); w.writerows(rows)
PY
python3 - <<'PY'
from pathlib import Path
from calibrate import load_absolute, ordinal_agreement, per_stratum
rows = load_absolute(Path("absolute_gold_v2.csv"))
print("Aggregate:", ordinal_agreement(rows)["spearman_rho"])
for s, v in per_stratum(rows, ordinal_agreement).items():
    print(f"  {s}: rho={v['spearman_rho']} CI={v['spearman_ci95']}")
PY
```

**Expected:** the `safety_critical` ρ drops further, pushing the stratum deeper into "hidden regression" territory. Chapter 04's escalation criterion ("inter-rater agreement should itself be measured on a sub-sample") is what this models — if two humans disagree on safety-critical, re-labelling the safety-critical stratum with a stricter human is one signal the **rubric anchors** need rewriting before the judge is iterated. The fix lives in chapter 01, not in a bigger N.

### Step 10 — Write the one-page calibration doc

Chapter 04's "calibration document" has eight fields. Create `calibration.md` with the aggregate and per-stratum numbers from Step 6 filled in:

```markdown
# Calibration: {rubric_name}

1. **Rubric and version.** Pointer to rubric doc; prompt-template hash: `<stub-no-hash>`.
2. **Judge model.** STUB — synthetic fixture data for lab-02; no real judge.
3. **Gold set.** N=60 (ordinal); N=60 (pairwise). Stratified 20/20/20 across
   `routine`, `edge_case`, `safety_critical`. Source: `absolute_gold.csv`,
   `pairwise_gold.csv`.
4. **Rater(s).** Lab-fixture synthetic labels (no inter-rater agreement measured).
5. **Statistic and value.**
   - Absolute: Spearman rho = <value>; 95% bootstrap CI [<lo>, <hi>];
     per-stratum: routine <rho>, edge_case <rho>, safety_critical <rho>.
   - Pairwise: three-class Cohen's kappa = <value>; 95% CI [<lo>, <hi>];
     non-tie agreement = <value>; 95% CI [<lo>, <hi>].
6. **Decision.** Not calibrated: `safety_critical` stratum is below
   the aggregate by more than 0.2 on both rubrics; escalate per chapter
   04 to peer for variance decomposition before trusting the aggregate.
7. **Re-calibration triggers.** Judge model change; rubric prompt hash
   change; drift monitor replay statistic breach; calibration set age > 3 months.
8. **Escalation status.** Delegation contract pending with
   `model-evaluation-engineer` peer for the safety-critical stratum.
```

The point of writing it is that chapter-04 is explicit: *"a judge without a calibration number is a judge you cannot defend."* The written doc is the artefact mod-107 points to when asked.

### Step 11 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `python3 run.py` prints two blocks (absolute, pairwise) each with an aggregate one-liner, per-stratum breakdown, and chapter-04 reading checks.
- [ ] The aggregate CIs on both rubrics do **not** cover zero on the seed fixture — the judge is defendable at the aggregate.
- [ ] The **`safety_critical` stratum** is flagged as a hidden regression on at least one rubric — the aggregate is hiding it.
- [ ] Dropping N from 60 to 27 roughly **doubles** the CI width while barely moving the point estimate (step 8).
- [ ] You can state which chapter-04 escalation criterion a stratum shortfall triggers (safety-critical ordinal rubric with κ-like statistic materially below aggregate → variance decomposition at peer-role altitude).
- [ ] You can state the four-step order chapter 04 prescribes when a calibration fails: **rewrite rubric anchors → add missing bias controls → switch judge tier → escalate to peer** — not "run more samples until it looks better".

## Where this hands off

- `calibrate.py` is the shape exercise-04 asks you to productionise against your real surface and your real 50–100-item hand-labelled gold set — with scipy bootstrap CIs and the same per-stratum breakdown.
- The `calibration.md` template is exactly the chapter-04 "calibration document" the mod-107 online-eval loop points at and the chapter-05 drift monitor consumes (the re-calibration triggers in field 7 are what chapter 05 enforces).
- The "aggregate hides a bad stratum" output is the signal chapter-04 says should trigger an escalation to `model-evaluation-engineer` — lab-02 does not write the delegation contract; exercise-04 does.
