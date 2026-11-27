# lab-01: Watch Position and Length Bias Controls Fire on a Deterministic Stub Judge

**Estimated duration:** 60–90 minutes
**Pairs with:** chapters 01 and 02 of this module.

## Objective

Internalise the chapter-02 vocabulary — **position-bias flag rate**, **swap-and-average aggregation**, **length-normalised scoring**, **length correlation of raw scores** — by driving them through a **stub judge** whose biases you control exactly. Because the judge is deterministic and the biases are injected on purpose, you get to see each control do or fail to do its job with no confounds from a real model.

At the end you can read the three chapter-02 diagnostic metrics off a pipeline you built yourself and name, for each one, what it caught and what it did not.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-01 writes the rubrics from scratch against a real product surface.
- Exercise-02 subjects *your* pairwise rubric to the same controls against **real** judge API calls on ~30 pairs from your surface.
- Exercise-03 implements the absolute rubric in DeepEval / Braintrust / Promptfoo.
- Exercises 04 and 05 handle calibration and drift.

This lab gives you a stubbed, position-biased, length-biased judge so the controls fire visibly and attribution is unambiguous — the mechanical skill labs-02 and -03, plus exercise-02, assume you have.

## Prerequisites

- Chapters 01 (rubric anatomy) and 02 (biases + controls) read.
- Python 3.10+ on `$PATH` (`python3 --version`).
- A working directory you can write files into. **No internet, no model API keys, no Docker** — the judge is a ~40-line pure-Python function that emits a bias on purpose.

## Layout you will build

```
mod-104-lab-01/
  pairs.jsonl            # 20 pairwise items with known "ground-truth" winners
  stub_judge.py          # the deterministic biased judge
  pipeline.py            # swap-and-average + length-adjust + diagnostics
  run.py                 # CLI driver
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-104-lab-01 && cd mod-104-lab-01
```

Confirm you are in an empty directory:

```bash
ls -a
```

**Expected:** `.`, `..`.

### Step 2 — Seed `pairs.jsonl`

Twenty (input, candidate A, candidate B, true_winner) rows. The `true_winner` column is what a perfect, unbiased judge should report; the stub judge in step 3 will systematically disagree with it in ways the chapter-02 controls should correct.

The dataset is stratified by two properties:

- **Quality split.** Ten pairs where candidate A is genuinely better, ten where B is genuinely better.
- **Length split.** Ten pairs where the *losing* candidate is roughly 3× the length of the winner (this is the length-bias trap); ten pairs where both candidates have similar length.

Create `pairs.jsonl`:

```json
{"id": "p01", "a": "The order ships Monday per the policy doc.", "b": "The order ships Monday per the policy doc, confirmed by document 12 paragraph 3 and cross-checked against the shipping schedule table and the customer's order record; additional context about the shipping window, the carrier's stated SLA, and the fallback escalation path is also summarised here in detail.", "true_winner": "A", "note": "A is correct, concise; B is correct but padded."}
{"id": "p02", "a": "Refund processed on 2024-03-14 per policy §4.1.", "b": "Refund processed on 2024-03-14 per policy §4.1; additional background on refund policy, historical refund trends over the last six quarters, and the typical resolution window across different payment methods is provided here as context so the customer has everything they might need.", "true_winner": "A", "note": "A is correct, concise; B is correct but padded."}
{"id": "p03", "a": "The policy PDF does not cover international shipping to this country.", "b": "The policy PDF does not cover international shipping to this country, and in addition we have cross-referenced every available international carrier agreement, the customs schedule, and the exemption registry to confirm no in-scope answer exists; the escalation path is documented below in detail.", "true_winner": "A", "note": "A is correct, concise; B is correct but padded."}
{"id": "p04", "a": "Per §2, the account is locked after three failed logins.", "b": "Per §2, the account is locked after three failed logins; additional details including every security policy reference, every audit-log reference, and every recovery-path step are enumerated here at length for completeness.", "true_winner": "A", "note": "A is correct, concise; B is correct but padded."}
{"id": "p05", "a": "The device must be returned within 30 days per §7.", "b": "The device must be returned within 30 days per §7; a comprehensive overview of the entire return policy, including the exceptions, the carrier choices, the inspection criteria, and the refund timeline is also provided here in detail.", "true_winner": "A", "note": "A is correct, concise; B is correct but padded."}
{"id": "p06", "a": "Order 777 shipped on 2024-04-01.", "b": "Order 777 shipped on 2024-03-25. (fabricated date)", "true_winner": "A", "note": "A is correct; B invents a date not in retrieval."}
{"id": "p07", "a": "The warranty covers defects for 24 months per §5.", "b": "The warranty covers defects for 36 months per §5. (fabricated duration)", "true_winner": "A", "note": "A is correct; B invents a duration not in retrieval."}
{"id": "p08", "a": "Status: delivered on 2024-02-11.", "b": "Status: delivered on 2024-02-13. (off by two days)", "true_winner": "A", "note": "A is correct; B has a minor fabrication."}
{"id": "p09", "a": "Policy §9 requires a signature on delivery.", "b": "Policy §10 requires a signature on delivery. (wrong section)", "true_winner": "A", "note": "A is correct; B cites the wrong section."}
{"id": "p10", "a": "Return-shipping label will be emailed within 24 hours.", "b": "Return-shipping label will be emailed within 72 hours. (wrong SLA)", "true_winner": "A", "note": "A is correct; B invents a wrong SLA."}
{"id": "p11", "a": "The subscription renews annually on the anniversary date, per document 3; customers can adjust the renewal date via the billing portal and should expect an email reminder one week before the charge.", "b": "The subscription renews monthly on the anniversary date. (wrong cadence)", "true_winner": "B-should-lose-but-shorter", "note": "A is correct and detailed; B is wrong but short — tests length bias in the other direction."}
{"id": "p12", "a": "Per the policy PDF, international orders take 7–10 business days; the timing varies by destination and carrier, and the document lists the exceptions for island destinations and remote regions in its appendix.", "b": "International orders take 3–5 business days. (wrong range)", "true_winner": "B-should-lose-but-shorter", "note": "A is correct and detailed; B is wrong but short."}
{"id": "p13", "a": "The recorded delivery date is 2024-05-07 as shown on the shipping log attached to the ticket, cross-referenced against the carrier confirmation email on file.", "b": "The recorded delivery date is 2024-06-07. (off by a month)", "true_winner": "B-should-lose-but-shorter", "note": "A is correct and detailed; B is wrong but short."}
{"id": "p14", "a": "Order 444 shipped on 2024-07-02 per the shipping confirmation attached to the policy PDF; the document also notes the carrier and the tracking number for cross-reference.", "b": "Order 444 shipped on 2024-07-20. (wrong date)", "true_winner": "B-should-lose-but-shorter", "note": "A is correct and detailed; B is wrong but short."}
{"id": "p15", "a": "The refund window is 14 days per §4.1 of the policy PDF; the document further notes that the window begins on the delivery date, not the order date, and lists the three qualifying exceptions.", "b": "The refund window is 30 days per §4.1. (wrong duration)", "true_winner": "B-should-lose-but-shorter", "note": "A is correct and detailed; B is wrong but short."}
{"id": "p16", "a": "The ticket is escalated to tier-2 per policy §6.", "b": "The ticket is escalated to tier-3 per policy §6. (wrong tier)", "true_winner": "A", "note": "Similar length; A is correct, B invents."}
{"id": "p17", "a": "Per §8, two-factor authentication is required for admin roles.", "b": "Per §8, two-factor authentication is optional for admin roles. (wrong policy)", "true_winner": "A", "note": "Similar length; A is correct, B inverts policy."}
{"id": "p18", "a": "The contract expires on 2025-01-01 per the renewal schedule.", "b": "The contract expires on 2025-02-01 per the renewal schedule. (wrong date)", "true_winner": "A", "note": "Similar length; A is correct, B invents."}
{"id": "p19", "a": "The user's email of record is alice@example.com per the account page.", "b": "The user's email of record is alice@example.org per the account page. (wrong TLD)", "true_winner": "A", "note": "Similar length; A is correct, B invents."}
{"id": "p20", "a": "Order 909 was cancelled by the customer on 2024-08-15.", "b": "Order 909 was cancelled by the merchant on 2024-08-15. (wrong actor)", "true_winner": "A", "note": "Similar length; A is correct, B inverts actor."}
```

> For the ten "padded" pairs (p01–p05, where B is longer-but-correct, and p11–p15, where A is longer-and-correct): the `true_winner` column records the ground truth a careful human would assign — groundedness is equal when both are correct (so a fair judge would often `tie` the padded pairs), and groundedness is **A** when A is correct and B is not. The `B-should-lose-but-shorter` label on p11–p15 is a narrative tag; your code should treat the ground-truth winner as `A` (parsed below).

### Step 3 — Write the biased stub judge (`stub_judge.py`)

The judge takes two candidates, a "position" argument naming which candidate is in slot `first`, and returns a winner. **It is biased on purpose**: it prefers the longer candidate, and when lengths are tied it prefers the one in slot `first`. There is **no content understanding** — this is the worst-case bias contaminant with no confounds from real quality.

```python
# stub_judge.py
"""Deterministic biased pairwise judge.

This is a *stub*: no model API, no content understanding. It simulates
the two worst-case biases chapter 02 documents (position + length) at
their maximum. A real judge has weaker biases and some content signal;
see exercise-02 for the real-model version. The point of the stub is
that the controls in pipeline.py have something unambiguous to correct.
"""

from __future__ import annotations

from typing import Literal

Winner = Literal["first", "second", "tie"]


def judge(first: str, second: str) -> dict:
    """Return {"winner": "first"|"second"|"tie", "rationale": str}.

    Bias model (all injected on purpose):
      1. If one candidate is > 1.5x the length of the other, prefer the
         longer one. (Length bias, strong.)
      2. Otherwise, prefer the candidate in the `first` slot. (Position
         bias, strong — the chapter-02 canonical case.)
      3. Lengths within 1% of each other → tie.
    """
    len_f, len_s = len(first), len(second)
    if min(len_f, len_s) == 0:
        return {"winner": "tie", "rationale": "empty candidate"}
    ratio = max(len_f, len_s) / min(len_f, len_s)
    if ratio > 1.5:
        winner = "first" if len_f > len_s else "second"
        return {"winner": winner, "rationale": f"longer ({max(len_f,len_s)} > {min(len_f,len_s)} chars)"}
    if ratio < 1.01:
        return {"winner": "tie", "rationale": "near-identical length"}
    return {"winner": "first", "rationale": "position bias (lengths close)"}
```

### Step 4 — Write the pipeline (`pipeline.py`)

The pipeline implements the chapter-02 controls around the stub judge:

- `pairwise_raw` runs the judge **once** with A in `first` — the naïve baseline.
- `pairwise_swap_and_average` runs the judge **twice** with swapped order and aggregates per chapter 02.
- `length_adjust` fits a tiny logistic-style regression of win probability against the length *difference* (standard-library numerics only; no scipy needed) and reports the length-adjusted win rate.
- `diagnostics` reports **position-bias flag rate**, **length correlation of raw scores**, and **swap-vs-raw agreement rate**.

```python
# pipeline.py
"""Chapter-02 bias-control pipeline wrapped around stub_judge.judge.

Three mitigations, three diagnostics, three attributes per judgement
(matching the mod-102 EVALUATOR-span attribute shape):
  app.judge_swap_order          — 'ab' or 'ba'
  app.judge_position_bias_flag  — did the two orders disagree
  app.judge_candidate_a_len     — chars in candidate A
  app.judge_candidate_b_len     — chars in candidate B
"""

from __future__ import annotations

import json
import math
from pathlib import Path
from typing import Callable

from stub_judge import judge as _judge

JudgeFn = Callable[[str, str], dict]


# --- raw / baseline --------------------------------------------------------

def pairwise_raw(a: str, b: str, judge: JudgeFn = _judge) -> dict:
    """One call, A in slot `first`. The naïve baseline the chapter rejects."""
    out = _normalise(judge(a, b), order="ab")
    return {
        "winner": out["winner"],              # "A" | "B" | "tie"
        "position_bias_flag": None,           # unknown — only one order run
        "app.judge_swap_order": "ab",
        "app.judge_candidate_a_len": len(a),
        "app.judge_candidate_b_len": len(b),
        "rationales": [out["rationale"]],
    }


# --- swap-and-average ------------------------------------------------------

def pairwise_swap_and_average(a: str, b: str, judge: JudgeFn = _judge) -> dict:
    """Two calls, swapped order. Disagreement collapses to tie."""
    ab = _normalise(judge(a, b), order="ab")
    ba = _normalise(judge(b, a), order="ba")
    if ab["winner"] == ba["winner"] and ab["winner"] in ("A", "B"):
        return {
            "winner": ab["winner"],
            "position_bias_flag": False,
            "app.judge_swap_order": "ab+ba",
            "app.judge_candidate_a_len": len(a),
            "app.judge_candidate_b_len": len(b),
            "rationales": [ab["rationale"], ba["rationale"]],
        }
    return {
        "winner": "tie",
        "position_bias_flag": ab["winner"] != ba["winner"],
        "app.judge_swap_order": "ab+ba",
        "app.judge_candidate_a_len": len(a),
        "app.judge_candidate_b_len": len(b),
        "rationales": [ab["rationale"], ba["rationale"]],
    }


def _normalise(raw: dict, *, order: str) -> dict:
    """Map {first/second/tie} to {A/B/tie} given the slot order."""
    if order == "ab":
        mapping = {"first": "A", "second": "B", "tie": "tie"}
    else:
        mapping = {"first": "B", "second": "A", "tie": "tie"}
    return {"winner": mapping[raw["winner"]], "rationale": raw["rationale"]}


# --- length-adjusted win rate ---------------------------------------------

def length_adjust(judgements: list[dict]) -> dict:
    """Fit a tiny logistic regression of P(A wins) on (len_a - len_b).

    Reports the length-adjusted win rate = predicted probability at
    length-difference = 0 (the Dubois et al. framing: "what would the
    win rate look like if both candidates had the same length?"). Pure
    stdlib — one-feature gradient descent is plenty for this shape.
    """
    X = [(j["app.judge_candidate_a_len"] - j["app.judge_candidate_b_len"]) for j in judgements]
    y = [1 if j["winner"] == "A" else 0 for j in judgements]  # ties dropped
    pairs = [(x, yi) for x, yi, j in zip(X, y, judgements) if j["winner"] != "tie"]
    if not pairs:
        return {"raw_win_rate_A": 0.0, "length_adjusted_win_rate_A": 0.0,
                "beta_length": 0.0, "intercept": 0.0, "n_used": 0}

    # Centre & scale X for numerical stability.
    xs = [x for x, _ in pairs]
    mean = sum(xs) / len(xs)
    std = max(1.0, math.sqrt(sum((x - mean) ** 2 for x in xs) / len(xs)))
    Xn = [(x - mean) / std for x in xs]
    yn = [yi for _, yi in pairs]

    w, b = 0.0, 0.0
    lr = 0.05
    for _ in range(2000):
        gw, gb = 0.0, 0.0
        for x, yi in zip(Xn, yn):
            p = 1.0 / (1.0 + math.exp(-(w * x + b)))
            gw += (p - yi) * x
            gb += (p - yi)
        w -= lr * gw / len(Xn)
        b -= lr * gb / len(Xn)

    # Predicted P(A wins) when len_a - len_b == 0 → Xn = (0 - mean)/std.
    x_at_zero = (0 - mean) / std
    p_at_zero = 1.0 / (1.0 + math.exp(-(w * x_at_zero + b)))

    raw_rate = sum(yn) / len(yn)
    # De-normalise beta back to original units for reporting.
    beta_raw = w / std
    intercept_raw = b - w * mean / std
    return {
        "raw_win_rate_A": round(raw_rate, 4),
        "length_adjusted_win_rate_A": round(p_at_zero, 4),
        "beta_length": round(beta_raw, 6),
        "intercept": round(intercept_raw, 4),
        "n_used": len(yn),
    }


# --- diagnostics -----------------------------------------------------------

def diagnostics(raw_judgements: list[dict], swap_judgements: list[dict]) -> dict:
    """Chapter-02 three-metric diagnostic block."""
    n = len(swap_judgements)
    flag_rate = sum(1 for j in swap_judgements if j["position_bias_flag"]) / max(1, n)

    # Length correlation of RAW winner (coded +1 if A wins, -1 if B, 0 tie)
    # against length difference (len_A - len_B).
    coded = []
    len_diff = []
    for j in raw_judgements:
        coded.append({"A": 1, "B": -1, "tie": 0}[j["winner"]])
        len_diff.append(j["app.judge_candidate_a_len"] - j["app.judge_candidate_b_len"])
    corr = _pearson(coded, len_diff)

    # Swap-vs-raw agreement: share of items where swap-and-average returned
    # the same non-tie winner as the raw baseline.
    agree = sum(
        1 for r, s in zip(raw_judgements, swap_judgements)
        if r["winner"] == s["winner"] and r["winner"] != "tie"
    ) / max(1, n)

    return {
        "position_bias_flag_rate": round(flag_rate, 4),
        "length_corr_raw_score": round(corr, 4),
        "swap_vs_raw_agreement": round(agree, 4),
        "n": n,
    }


def _pearson(xs: list[float], ys: list[float]) -> float:
    if not xs:
        return 0.0
    mx = sum(xs) / len(xs)
    my = sum(ys) / len(ys)
    num = sum((x - mx) * (y - my) for x, y in zip(xs, ys))
    dx = math.sqrt(sum((x - mx) ** 2 for x in xs))
    dy = math.sqrt(sum((y - my) ** 2 for y in ys))
    return num / (dx * dy) if dx > 0 and dy > 0 else 0.0


# --- CLI entry point -------------------------------------------------------

def load_pairs(path: Path) -> list[dict]:
    rows = []
    for line in path.read_text().splitlines():
        if line.strip():
            rows.append(json.loads(line))
    return rows


def ground_truth_winner(row: dict) -> str:
    # Both quality-split tags collapse to A — in this fixture A is the
    # ground-truth winner on every row. The tag is a narrative label.
    return "A"
```

### Step 5 — Write the CLI driver (`run.py`)

```python
# run.py
"""Run all three pipelines on pairs.jsonl and print a side-by-side table."""

import json
from pathlib import Path

from pipeline import (
    diagnostics,
    ground_truth_winner,
    length_adjust,
    load_pairs,
    pairwise_raw,
    pairwise_swap_and_average,
)


def main() -> int:
    rows = load_pairs(Path("pairs.jsonl"))
    raw = [pairwise_raw(r["a"], r["b"]) for r in rows]
    swap = [pairwise_swap_and_average(r["a"], r["b"]) for r in rows]

    truth = [ground_truth_winner(r) for r in rows]
    agree_raw = sum(1 for r, t in zip(raw, truth) if r["winner"] == t) / len(rows)
    agree_swap = sum(1 for s, t in zip(swap, truth) if s["winner"] == t) / len(rows)

    adj = length_adjust(raw)
    diag = diagnostics(raw, swap)

    print("## Agreement with ground truth")
    print(f"  raw (one order)        : {agree_raw:.3f}")
    print(f"  swap-and-average       : {agree_swap:.3f}")
    print()
    print("## Length adjustment on raw judgements")
    for k, v in adj.items():
        print(f"  {k:28s}: {v}")
    print()
    print("## Chapter-02 diagnostics")
    for k, v in diag.items():
        print(f"  {k:28s}: {v}")
    print()
    print("## Per-item breakdown (id, raw winner, swap winner, pos_flag, lenA, lenB)")
    for row, r, s in zip(rows, raw, swap):
        print(f"  {row['id']} raw={r['winner']:4s} swap={s['winner']:4s} "
              f"flag={str(s['position_bias_flag']):5s} "
              f"lenA={r['app.judge_candidate_a_len']:4d} "
              f"lenB={r['app.judge_candidate_b_len']:4d}")
    return 0


if __name__ == "__main__":
    import sys
    sys.exit(main())
```

### Step 6 — Run the baseline

```bash
python3 run.py
```

**Expected (high-level)**:

- **Agreement with ground truth, raw** is low — the stub judge's length + position biases push the winner toward the longer candidate (or toward `first` when lengths are close), which disagrees with the ground-truth `A` on exactly the pairs where B is longer-but-wrong (p06–p10, p11–p15 inverted direction, p16–p20 slot-dependent) or longer-but-only-padded (p01–p05).
- **Agreement with ground truth, swap-and-average** is higher *on the length-close pairs*, because the position bias cancels between the two orders. It is **not** higher on the padded / length-different pairs — swap-and-average does **not** fix length bias; the chapter-02 pipeline needs length adjustment for that.
- **`position_bias_flag_rate`** is substantially above zero — specifically, the fraction of pairs where the two candidates have similar lengths (p16–p20 and the length-close subset of p01–p15).
- **`length_corr_raw_score`** is strongly positive — raw winner tracks `len_A − len_B` because the stub prefers the longer slot. This is the chapter-02 "material non-zero" signal that says either length is a real quality axis on your surface (not the case here) or the judge is length-biased (which it is).
- **`length_adjusted_win_rate_A`** is closer to the ground-truth A-win rate than the raw rate — length adjustment extracts the content-conditional signal even from a judge whose only signal *is* length.

Read the exact numbers off the printed table; they are deterministic from the fixture.

### Step 7 — Flip one control off and confirm it mattered

Comment out the second call inside `pairwise_swap_and_average` so the pipeline degenerates to one order; re-run.

```bash
python3 - <<'PY'
import re
from pathlib import Path
src = Path("pipeline.py").read_text()
patched = src.replace(
    '    ba = _normalise(judge(b, a), order="ba")\n',
    '    ba = ab  # LAB STEP 7: disabled swap\n',
)
Path("pipeline.py").write_text(patched)
PY
python3 run.py
```

**Expected**: `position_bias_flag_rate` drops to **0.0** (there is no second order to disagree with), the swap-vs-raw agreement rises to **1.0**, and swap agreement with ground truth equals the raw agreement. The flag you lost is exactly the chapter-02 signal that keeps near-tied decisions from being flipped by position. Restore the swap:

```bash
python3 - <<'PY'
from pathlib import Path
src = Path("pipeline.py").read_text()
restored = src.replace(
    '    ba = ab  # LAB STEP 7: disabled swap\n',
    '    ba = _normalise(judge(b, a), order="ba")\n',
)
Path("pipeline.py").write_text(restored)
PY
```

### Step 8 — Flip the length adjustment off and confirm it mattered

Replace the `length_adjust` call in `run.py` with the raw win rate only, re-run, and read the "raw_win_rate_A" number without the adjusted one next to it. Observe that the raw rate on this fixture is **misleading on purpose** — the stub prefers the longer slot, so the raw rate is a function of the length distribution rather than content quality. Length adjustment is the control that extracts the content signal from a biased raw score; on a real pipeline, this is what the chapter-02 "length-controlled" number is.

### Step 9 — Attribute the residual errors

For each item where **swap-and-average still disagrees with the ground truth**, identify which chapter-02 bias is responsible. Expected pattern:

- **p01–p05** (B longer, both correct): swap-and-average will usually return the longer candidate (B) as winner because length bias dominates in both orders. Ground truth is `A` under our coding rule, so these count as misses. Length adjustment is the control that corrects this, *not* swap-and-average.
- **p11–p15** (A longer, A correct): swap-and-average returns A (correctly). Length adjustment on its own (if length were the only signal) would flatten the margin — but since A is both longer and correct, no fix is needed.
- **p16–p20** (similar length, A correct): swap-and-average fires exactly because the two orders disagree (position bias is the only signal when lengths are tied), so these collapse to `tie` under the chapter-02 rule — *honest* information, not a miss.

Write these three attribution lines into a `report.md` in the same directory. mod-109 reviewers will consume exactly this attribution shape when they audit near-tie decisions in production.

### Step 10 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `python3 run.py` prints three sections (ground-truth agreement, length adjustment, chapter-02 diagnostics) plus a per-item table.
- [ ] The **position-bias flag rate** is non-zero on the seed fixture and drops to zero when the swap is disabled (step 7).
- [ ] The **length correlation** on raw scores is materially non-zero (the stub prefers the longer candidate).
- [ ] The **length-adjusted win rate for A** is closer to the ground-truth A-win rate than the raw rate is.
- [ ] You can name, for each residual ground-truth miss under swap-and-average, which chapter-02 bias is responsible: **position**, **length**, or an **honest tie**.
- [ ] You can state why swap-and-average alone does *not* fix length bias (both orders still prefer the longer candidate), and why length adjustment alone does *not* fix position bias (it does not look at slot order).

## Where this hands off

- The three control entry points (`pairwise_raw`, `pairwise_swap_and_average`, `length_adjust`) map one-to-one onto the chapter-02 "judge runtime boundary" — exercise-02 productionises the same entry points against a real judge API and your own rubric.
- The three diagnostics (`position_bias_flag_rate`, `length_corr_raw_score`, `swap_vs_raw_agreement`) are the exact chapter-02 metrics chapter-05's drift monitor reads; lab-03 shows the drift monitor consuming a parallel block.
- The `app.judge_*` attributes emitted per judgement match the mod-102 EVALUATOR-span shape — in production they land on the trace, not the stdout table.
