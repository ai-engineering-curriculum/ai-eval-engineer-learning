# lab-03: Build the Per-Slice × Per-Metric Dashboard and Catch a Silent Regression

**Estimated duration:** 60–75 minutes
**Pairs with:** chapters 03 and 04 of this module.

## Objective

Make the chapter-04 **silent regression** catch mechanical. You assemble the per-slice × per-metric dashboard chapter 04 prescribes — eight rows (four triad + four diagnostics), one column per chapter-03 slice — from a canned per-item fixture. Then you flip the fixture to an "after" state in which the **triad aggregates on every slice stay within a 0.03 noise floor** while **noise sensitivity on the `distractor_heavy` slice drifts by paging-worthy magnitude**. The dashboard fires on the diagnostic cell before any triad cell moves — which is the chapter-04 argument for keeping the diagnostics around in the first place.

At the end you have (i) a tiny `dashboard.py` that produces the chapter-04 reporting shape with per-slice denominators, `not_applicable` cells, and alert states; (ii) the "before"/"after" fixtures that demonstrate the silent-regression pattern; and (iii) a short regression report that argues in the chapter-04 shape — numbers, denominators, which metric fired, which did not.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-01 runs the real triad metrics (RAGAS + TruLens) against your real surface with a real judge API.
- Exercise-03 grows the real product-shaped gold set with ~150–300 items and the three signature slices.
- Exercise-04 runs real RAGAS `NoiseSensitivity` / `ContextUtilization` / `ResponseCompleteness` and a mod-104 refusal-quality rubric against the exercise-03 gold set, seeds a real silent regression on your retriever, and wires the dashboard to a live observability backend.
- Lab-01 and lab-02 built the retrieval-only metrics and the attribution decision tree against canned fixtures.

This lab gives you *canned per-item scores* for every metric on every slice — the kind of output you would get *after* exercise-04 runs — and drills the aggregation / thresholding / alerting logic that turns those scores into the dashboard-and-alert shape mod-107 pages on. The scores are hand-authored: they preserve chapter-04's silent-regression shape so the paging logic has an unambiguous signal.

## Prerequisites

- Chapters 03 and 04 read.
- Python 3.10+ on `$PATH`. Stdlib only (`statistics` from stdlib covers the means).
- A working directory you can write files into. **No internet, no model API keys, no RAGAS install** — the per-item scores are canned in CSV so this lab runs offline.

## Layout you will build

```
mod-105-lab-03/
  fixtures/
    before/
      scores.csv          # per-item scores for all 8 metrics, before the regression
    after/
      scores.csv          # per-item scores after the seeded silent regression
  dashboard.py            # the aggregator + alert checker
  run.py                  # CLI driver — prints the per-slice × per-metric table
```

The CSV schema (common to both files):

```
query_id, slice, faithfulness, answer_relevancy, context_precision, context_recall,
noise_sensitivity, context_utilisation, completeness, refusal_quality
```

Where `refusal_quality` is one of `correct_refusal | false_answer | false_refusal | correct_answer` (applied to `out_of_scope` items only), and all numeric cells can be `n/a` for not-applicable combinations per chapter-04's metric table (noise sensitivity on out-of-scope, refusal quality on in-scope, etc.).

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-105-lab-03/fixtures/before mod-105-lab-03/fixtures/after && cd mod-105-lab-03
```

Confirm the layout:

```bash
ls fixtures/
```

**Expected:** `after`, `before`.

### Step 2 — Author the `before` fixture

Twenty-four items — six per slice — across the four slices chapter 03 tags. For each item, nine columns of scores. The scores are hand-authored to preserve the chapter-04 metric table:

- `noise_sensitivity` is `n/a` on `out_of_scope` (no reference answer semantics) and populated elsewhere.
- `context_utilisation` is `n/a` on `out_of_scope` (refusal doesn't "use" chunks) and populated elsewhere.
- `completeness` is `n/a` on `out_of_scope` (no reference answer) and populated elsewhere.
- `refusal_quality` is `n/a` on in-scope items; populated on `out_of_scope` as one of the four categorical values from chapter 04.

Write `fixtures/before/scores.csv`:

```csv
query_id,slice,faithfulness,answer_relevancy,context_precision,context_recall,noise_sensitivity,context_utilisation,completeness,refusal_quality
q01,in_scope_single_chunk,0.95,0.92,0.95,1.00,0.05,0.75,0.95,n/a
q02,in_scope_single_chunk,0.92,0.95,0.98,1.00,0.03,0.80,0.98,n/a
q03,in_scope_single_chunk,0.95,0.90,0.95,1.00,0.05,0.78,0.95,n/a
q04,in_scope_single_chunk,0.90,0.92,0.95,1.00,0.08,0.72,0.92,n/a
q05,in_scope_single_chunk,0.92,0.90,0.95,1.00,0.05,0.75,0.95,n/a
q06,in_scope_single_chunk,0.95,0.92,0.95,1.00,0.03,0.80,0.98,n/a
q07,in_scope_multi_chunk,0.88,0.90,0.82,0.80,0.10,0.78,0.85,n/a
q08,in_scope_multi_chunk,0.90,0.88,0.80,0.78,0.08,0.75,0.82,n/a
q09,in_scope_multi_chunk,0.85,0.90,0.85,0.82,0.12,0.80,0.88,n/a
q10,in_scope_multi_chunk,0.88,0.88,0.80,0.78,0.10,0.75,0.82,n/a
q11,in_scope_multi_chunk,0.90,0.92,0.82,0.80,0.08,0.78,0.85,n/a
q12,in_scope_multi_chunk,0.88,0.88,0.80,0.78,0.10,0.75,0.82,n/a
q13,distractor_heavy,0.88,0.90,0.72,0.80,0.09,0.72,0.88,n/a
q14,distractor_heavy,0.85,0.88,0.70,0.78,0.08,0.70,0.85,n/a
q15,distractor_heavy,0.90,0.92,0.75,0.82,0.07,0.75,0.90,n/a
q16,distractor_heavy,0.85,0.88,0.72,0.80,0.10,0.72,0.88,n/a
q17,distractor_heavy,0.88,0.90,0.70,0.78,0.08,0.70,0.85,n/a
q18,distractor_heavy,0.85,0.88,0.72,0.80,0.10,0.72,0.88,n/a
q19,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
q20,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
q21,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
q22,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
q23,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,false_answer
q24,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
```

Per-slice means on the before fixture (hand-check):

| slice (n=6) | faithfulness | answer_relevancy | ctx_precision | ctx_recall | noise_sensitivity | ctx_utilisation | completeness |
|---|---|---|---|---|---|---|---|
| in_scope_single_chunk | ≈ 0.932 | ≈ 0.918 | ≈ 0.955 | 1.000 | ≈ 0.048 | ≈ 0.767 | ≈ 0.955 |
| in_scope_multi_chunk | ≈ 0.882 | ≈ 0.893 | ≈ 0.815 | ≈ 0.793 | ≈ 0.097 | ≈ 0.768 | ≈ 0.840 |
| distractor_heavy | ≈ 0.868 | ≈ 0.893 | ≈ 0.718 | ≈ 0.797 | ≈ 0.087 | ≈ 0.718 | ≈ 0.873 |

Refusal-quality breakdown on `out_of_scope` (n=6): 5 `correct_refusal`, 1 `false_answer`, 0 `false_refusal`. Refusal **recall** (fraction of true out-of-scope that refused correctly) = 5/6 ≈ 0.833.

### Step 3 — Author the `after` fixture (the seeded silent regression)

The seeded regression is chapter-04's canonical silent-regression shape: the retriever starts surfacing a plausible-but-irrelevant extra chunk on each `distractor_heavy` item (chapter 04's "retriever over-fetches on the distractor slice" case). The generator does not yet shift much on faithfulness or answer relevancy — the incorrect-claim rate is low and the claim decomposition is coarse — but noise sensitivity climbs on the affected slice because the generator now has an extra irrelevant chunk to lean on.

Preserving this shape in the fixture:

- **`in_scope_single_chunk` / `in_scope_multi_chunk` / `out_of_scope`**: every score moves by at most 0.01–0.02 in either direction — within the 0.03 noise floor.
- **`distractor_heavy`**: faithfulness moves at most 0.02 (within noise floor); context precision moves at most 0.02 (within noise floor); **noise sensitivity climbs from a slice mean of ≈ 0.087 to ≈ 0.170** — this is the chapter-04 signal that pages.
- **Out-of-scope refusal mix** is unchanged (5 correct, 1 false) — out-of-scope regressions are a separate paging path (chapter 04's refusal-recall diagnostic).

Write `fixtures/after/scores.csv`:

```csv
query_id,slice,faithfulness,answer_relevancy,context_precision,context_recall,noise_sensitivity,context_utilisation,completeness,refusal_quality
q01,in_scope_single_chunk,0.95,0.92,0.95,1.00,0.05,0.75,0.95,n/a
q02,in_scope_single_chunk,0.92,0.95,0.98,1.00,0.03,0.80,0.98,n/a
q03,in_scope_single_chunk,0.94,0.90,0.95,1.00,0.05,0.78,0.95,n/a
q04,in_scope_single_chunk,0.90,0.92,0.95,1.00,0.08,0.72,0.92,n/a
q05,in_scope_single_chunk,0.92,0.90,0.95,1.00,0.05,0.75,0.95,n/a
q06,in_scope_single_chunk,0.94,0.92,0.95,1.00,0.03,0.80,0.98,n/a
q07,in_scope_multi_chunk,0.88,0.90,0.82,0.80,0.10,0.78,0.85,n/a
q08,in_scope_multi_chunk,0.89,0.88,0.80,0.78,0.08,0.75,0.82,n/a
q09,in_scope_multi_chunk,0.85,0.90,0.85,0.82,0.12,0.80,0.88,n/a
q10,in_scope_multi_chunk,0.88,0.88,0.80,0.78,0.10,0.75,0.82,n/a
q11,in_scope_multi_chunk,0.89,0.92,0.82,0.80,0.08,0.78,0.85,n/a
q12,in_scope_multi_chunk,0.88,0.88,0.80,0.78,0.10,0.75,0.82,n/a
q13,distractor_heavy,0.87,0.89,0.71,0.80,0.18,0.72,0.88,n/a
q14,distractor_heavy,0.84,0.87,0.70,0.78,0.16,0.70,0.85,n/a
q15,distractor_heavy,0.89,0.91,0.74,0.82,0.14,0.75,0.90,n/a
q16,distractor_heavy,0.84,0.88,0.72,0.80,0.19,0.72,0.88,n/a
q17,distractor_heavy,0.88,0.90,0.70,0.78,0.16,0.70,0.85,n/a
q18,distractor_heavy,0.84,0.88,0.71,0.80,0.19,0.72,0.88,n/a
q19,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
q20,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
q21,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
q22,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
q23,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,false_answer
q24,out_of_scope,n/a,n/a,n/a,n/a,n/a,n/a,n/a,correct_refusal
```

Hand-check the slice means and deltas on `distractor_heavy` (the paging-critical slice):

- Faithfulness: before ≈ 0.868, after ≈ 0.860 → delta ≈ −0.008, well within 0.03 noise floor.
- Context precision: before ≈ 0.718, after ≈ 0.713 → delta ≈ −0.005, within floor.
- **Noise sensitivity**: before ≈ 0.087, after ≈ 0.170 → delta ≈ **+0.083** on a 0.03 floor → fires.
- Context utilisation: before ≈ 0.718, after ≈ 0.718 → flat.

The regression is real and noise-sensitivity-only on this slice. A dashboard that only reports the triad would show every cell on the slice within noise floor. Chapter 04's point.

### Step 4 — Write `dashboard.py`

```python
# dashboard.py
"""Chapter-04 per-slice × per-metric dashboard with alert states.

Rows: eight metrics (four triad + four diagnostics).
Columns: the four chapter-03 slices.
Cells: mean (n=denominator), plus an alert if the metric moved outside
       the per-metric per-slice floor compared to a baseline run.

Not-applicable combinations (chapter 04's metric table):
  noise_sensitivity, completeness → n/a on out_of_scope (need ref answer)
  context_utilisation             → n/a on out_of_scope (refusal uses no chunks)
  refusal_quality                 → n/a on in-scope slices
"""

from __future__ import annotations

import csv
import statistics
from collections import defaultdict
from pathlib import Path


SLICES = ["in_scope_single_chunk", "in_scope_multi_chunk", "distractor_heavy", "out_of_scope"]
NUMERIC_METRICS = [
    "faithfulness", "answer_relevancy", "context_precision", "context_recall",
    "noise_sensitivity", "context_utilisation", "completeness",
]

# Per-metric, per-slice applicability. True = compute a mean; False = emit 'n/a'.
APPLICABLE = {
    ("faithfulness",       "out_of_scope"): False,
    ("answer_relevancy",   "out_of_scope"): False,
    ("context_precision",  "out_of_scope"): False,
    ("context_recall",     "out_of_scope"): False,
    ("noise_sensitivity",  "out_of_scope"): False,
    ("context_utilisation","out_of_scope"): False,
    ("completeness",       "out_of_scope"): False,
}


def load_scores(path: Path) -> list[dict]:
    rows = []
    with path.open() as f:
        for row in csv.DictReader(f):
            parsed = {"query_id": row["query_id"], "slice": row["slice"]}
            for m in NUMERIC_METRICS:
                v = row.get(m)
                parsed[m] = None if v in ("n/a", "", None) else float(v)
            parsed["refusal_quality"] = row.get("refusal_quality") or "n/a"
            rows.append(parsed)
    return rows


def numeric_table(rows: list[dict]) -> dict:
    """Return {(metric, slice): {"mean": float|None, "n": int}}."""
    cells = {}
    bucket = defaultdict(list)
    for r in rows:
        for m in NUMERIC_METRICS:
            if APPLICABLE.get((m, r["slice"]), True) and r[m] is not None:
                bucket[(m, r["slice"])].append(r[m])
    for m in NUMERIC_METRICS:
        for s in SLICES:
            if not APPLICABLE.get((m, s), True):
                cells[(m, s)] = {"mean": None, "n": 0, "applicable": False}
                continue
            vals = bucket.get((m, s), [])
            cells[(m, s)] = {
                "mean": round(statistics.fmean(vals), 4) if vals else None,
                "n": len(vals),
                "applicable": True,
            }
    return cells


def refusal_quality_summary(rows: list[dict]) -> dict:
    """Four-label refusal-quality tally on out-of-scope items.

    Chapter 04 names refusal recall (correct_refusal / all true OOS) as
    the paging-critical number; refusal precision would need a sample of
    in-scope-items-that-got-refused, which lives in a different slice.
    """
    counts = defaultdict(int)
    for r in rows:
        if r["slice"] == "out_of_scope":
            counts[r["refusal_quality"]] += 1
    total_oos = sum(counts.values())
    correct = counts.get("correct_refusal", 0)
    false_answer = counts.get("false_answer", 0)
    return {
        "n": total_oos,
        "counts": dict(counts),
        "refusal_recall": round(correct / total_oos, 4) if total_oos else None,
        "false_answer_rate": round(false_answer / total_oos, 4) if total_oos else None,
    }


def diff_cells(before: dict, after: dict, floors: dict) -> dict:
    """For each (metric, slice) cell, compute delta and paging state.

    `floors[(metric, slice)]` is the material-regression threshold per
    chapter 04 — different metrics tolerate different per-slice shifts.
    A missing floor falls back to the global default 0.03.
    """
    diffs = {}
    for key, a_cell in after.items():
        b_cell = before.get(key, {})
        if not a_cell.get("applicable", True) or b_cell.get("mean") is None or a_cell.get("mean") is None:
            diffs[key] = {"delta": None, "state": "n/a"}
            continue
        delta = round(a_cell["mean"] - b_cell["mean"], 4)
        floor = floors.get(key, 0.03)
        # Direction-aware alerting: faithfulness / recall / completeness /
        # utilisation / refusal_recall page on DROP; noise_sensitivity and
        # false_answer_rate page on CLIMB. The dashboard flags any move
        # whose magnitude >= floor; the on-call uses the sign per chapter 04.
        state = "alert" if abs(delta) >= floor else "ok"
        diffs[key] = {"delta": delta, "state": state, "floor": floor}
    return diffs
```

### Step 5 — Write the CLI driver (`run.py`)

```python
# run.py
"""Prints the chapter-04 per-slice × per-metric dashboard, then the delta vs baseline."""

import sys
from pathlib import Path

from dashboard import (
    NUMERIC_METRICS, SLICES, diff_cells, load_scores, numeric_table, refusal_quality_summary,
)


# Per-metric per-slice floors. Chapter 04 is explicit these are
# surface-specific; the defaults here match chapter-04's worked example
# where noise_sensitivity on distractor_heavy carries a tighter per-slice
# floor than the aggregate.
FLOORS = {
    ("noise_sensitivity", "distractor_heavy"):  0.05,
    ("noise_sensitivity", "in_scope_multi_chunk"): 0.05,
    # Everything else falls back to 0.03.
}


def print_table(title: str, cells: dict) -> None:
    print(f"\n## {title}")
    header = f"{'metric':22s} | " + " | ".join(f"{s:22s}" for s in SLICES)
    print(header)
    print("-" * len(header))
    for m in NUMERIC_METRICS:
        row_parts = [f"{m:22s}"]
        for s in SLICES:
            cell = cells[(m, s)]
            if not cell["applicable"]:
                row_parts.append(f"{'n/a':22s}")
            elif cell["mean"] is None:
                row_parts.append(f"{'(no data)':22s}")
            else:
                row_parts.append(f"{cell['mean']:.4f} (n={cell['n']})".ljust(22))
        print(" | ".join(row_parts))


def print_diffs(diffs: dict) -> None:
    print("\n## Deltas vs baseline (alert = |delta| >= per-slice floor)")
    header = f"{'metric':22s} | " + " | ".join(f"{s:22s}" for s in SLICES)
    print(header)
    print("-" * len(header))
    for m in NUMERIC_METRICS:
        row_parts = [f"{m:22s}"]
        for s in SLICES:
            d = diffs[(m, s)]
            if d["state"] == "n/a":
                row_parts.append(f"{'n/a':22s}")
                continue
            tag = "ALERT" if d["state"] == "alert" else "ok"
            row_parts.append(f"delta={d['delta']:+.4f} [{tag}]".ljust(22))
        print(" | ".join(row_parts))


def main() -> int:
    before = load_scores(Path(sys.argv[1]))
    after = load_scores(Path(sys.argv[2]))

    before_cells = numeric_table(before)
    after_cells = numeric_table(after)

    print("# Chapter-04 per-slice × per-metric dashboard\n")
    print_table("Before (baseline)", before_cells)
    print_table("After (candidate)", after_cells)

    diffs = diff_cells(before_cells, after_cells, FLOORS)
    print_diffs(diffs)

    print("\n## Refusal-quality (out_of_scope slice only)")
    rb = refusal_quality_summary(before)
    ra = refusal_quality_summary(after)
    print(f"  before: counts={rb['counts']}, refusal_recall={rb['refusal_recall']}, false_answer_rate={rb['false_answer_rate']}")
    print(f"  after : counts={ra['counts']}, refusal_recall={ra['refusal_recall']}, false_answer_rate={ra['false_answer_rate']}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### Step 6 — Run the dashboard on `before` only (baseline health)

```bash
python3 run.py fixtures/before/scores.csv fixtures/before/scores.csv
```

Running the baseline against itself prints the "before" and "after" tables identically and every delta as `ok`. This is the sanity check: every cell has a defined mean (or `n/a` on a non-applicable combination), every denominator is `n=6`, and no alert fires.

**Expected (abbreviated):**

- `faithfulness` cell `(distractor_heavy, before)` ≈ `0.8683 (n=6)`.
- `noise_sensitivity` cell `(distractor_heavy, before)` ≈ `0.0867 (n=6)`.
- `faithfulness` cell `(out_of_scope, *)` prints `n/a`.
- Delta table: every cell `ok` or `n/a`.
- Refusal-quality: `refusal_recall ≈ 0.833, false_answer_rate ≈ 0.167`.

If the "before vs before" delta table shows any `ALERT` state, the floors or the mean computation is wrong — fix before step 7.

### Step 7 — Run the dashboard on `before → after`

```bash
python3 run.py fixtures/before/scores.csv fixtures/after/scores.csv
```

**Expected:**

- **Triad rows** (`faithfulness`, `answer_relevancy`, `context_precision`, `context_recall`) across every slice: deltas within `±0.01` → **every cell `ok`**.
- **`noise_sensitivity`** row:
  - `in_scope_single_chunk`: delta ≈ `0.0000`, `ok`.
  - `in_scope_multi_chunk`: delta ≈ `0.0000`, `ok`.
  - `distractor_heavy`: delta ≈ **`+0.0833`** on a 0.05 floor → **`ALERT`**.
  - `out_of_scope`: `n/a`.
- `context_utilisation`, `completeness` rows: every cell `ok`.
- Refusal-quality: unchanged before vs after (`refusal_recall ≈ 0.833` both runs).

Read the shape: **every triad cell is green on every slice**. One diagnostic cell — `noise_sensitivity` on `distractor_heavy` — is red. This is chapter 04's worked silent-regression catch, on a canned fixture, in code.

### Step 8 — Flip the diagnostic off; confirm it mattered

Temporarily remove `noise_sensitivity` from the dashboard's metric list and re-run:

```bash
python3 - <<'PY'
from pathlib import Path
src = Path("dashboard.py").read_text()
patched = src.replace(
    '"noise_sensitivity", "context_utilisation", "completeness",',
    '"context_utilisation", "completeness",',
)
Path("dashboard.py").write_text(patched)
PY
python3 run.py fixtures/before/scores.csv fixtures/after/scores.csv
```

**Expected:** the delta table now has **zero `ALERT` cells across every slice**. The regression has become genuinely silent. This is the chapter-04 point: without the diagnostic metric, the regression ships. Restore the diagnostic:

```bash
python3 - <<'PY'
from pathlib import Path
src = Path("dashboard.py").read_text()
restored = src.replace(
    '"context_utilisation", "completeness",',
    '"noise_sensitivity", "context_utilisation", "completeness",',
)
Path("dashboard.py").write_text(restored)
PY
```

### Step 9 — Seed a second silent regression: refusal-quality drift

Chapter 04's other silent-regression pathway is a **refusal-recall drop on `out_of_scope`** while every triad aggregate stays flat. Simulate it by flipping two of the six `out_of_scope` items from `correct_refusal` to `false_answer`:

```bash
python3 - <<'PY'
from pathlib import Path
# Start from the clean `before` fixture so only the refusal change is seeded —
# we want this scenario to isolate the refusal-quality signal.
rows = Path("fixtures/before/scores.csv").read_text().splitlines()
patched = []
flips = 0
for line in rows:
    if line.startswith("q19,") or line.startswith("q20,"):
        patched.append(line.replace("correct_refusal", "false_answer"))
        flips += 1
    else:
        patched.append(line)
Path("fixtures/after_oos").mkdir(exist_ok=True)
Path("fixtures/after_oos/scores.csv").write_text("\n".join(patched) + "\n")
print(f"flipped {flips} rows")
PY
python3 run.py fixtures/before/scores.csv fixtures/after_oos/scores.csv
```

**Expected:**

- Every numeric cell `ok` on every slice (the per-item in-scope / distractor_heavy numbers are identical to the baseline here — only the out-of-scope refusal labels changed).
- Refusal-quality: `before: refusal_recall ≈ 0.833, false_answer_rate ≈ 0.167`; `after: refusal_recall = 0.500, false_answer_rate = 0.500`.

Chapter 04's reading: a 33-point drop in refusal recall with every other dashboard cell green. Without the refusal-quality diagnostic, the regression is literally un-measurable because faithfulness and answer relevancy are `n/a` on out-of-scope by chapter-01 design. The refusal diagnostic is what gives the out-of-scope slice any paging surface at all.

### Step 10 — Write the regression report

Open `regression-report.md` and write a 200–350-word report in the shape mod-107's online-eval loop uses for a diagnostic-only page. The structure mirrors chapter 05's handoff template:

```
Regression: <metric> on <slice>, <magnitude>, <direction>
Sample size: <n items scored>, per-slice
Attribution: diagnostic-only (triad aggregates within <floor>)
Evidence:
  - noise_sensitivity on distractor_heavy: <before> → <after>  (floor=<floor>)
  - faithfulness on distractor_heavy:     <before> → <after>   (delta within floor)
  - context_precision on distractor_heavy: <before> → <after>  (delta within floor)
  - every other slice × metric cell:       within floor
Representative items (3):
  - <query_id>, slice, noise_sensitivity delta, candidate reply pattern
Suspected driver (informed guess, not diagnosis):
  - retriever over-fetch: an extra plausibly-related chunk is reaching
    the generator; the generator leans on it; noise sensitivity rises
    before faithfulness does (chapter 04's lead indicator case)
Requested response:
  - acknowledge by <timeframe>; chapter-02 attribution tree run on the
    distractor_heavy slice first (retriever-side suspected) before any
    rubric anchor change on this side
```

The report is the deliverable chapter 04 insists on: *a stakeholder unfamiliar with the module can read it and understand why the diagnostics exist.*

### Step 11 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `python3 run.py fixtures/before/scores.csv fixtures/before/scores.csv` prints zero `ALERT` cells.
- [ ] `python3 run.py fixtures/before/scores.csv fixtures/after/scores.csv` prints **exactly one** `ALERT` cell: `noise_sensitivity` on `distractor_heavy`, delta ≈ `+0.08`.
- [ ] The four triad rows are `ok` on every slice for the `before → after` run.
- [ ] Non-applicable cells (`faithfulness` on `out_of_scope`, `refusal_quality` as a numeric row, etc.) print `n/a` — they do not degrade to `0.0` and they do not trigger an alert.
- [ ] Removing `noise_sensitivity` from the metric list (step 8) hides the regression entirely — the dashboard is green.
- [ ] The second seeded regression (step 9) moves `refusal_recall` on `out_of_scope` from ≈ 0.833 to 0.500 while every other cell stays `ok` — and you can explain why this is only visible via the chapter-04 refusal-quality diagnostic and never via faithfulness / answer relevancy.
- [ ] You can name, for each of the four diagnostics in the dashboard, the chapter-04 failure mode it catches: noise sensitivity (grounded-but-wrong-chunk), context utilisation (over-fetching / under-using), completeness (half-answered multi-part), refusal quality (false-positive answer on out-of-scope).
- [ ] You can explain the chapter-04 paging rule: a diagnostic-only regression is still a regression, and the alert channel for the diagnostic row is the same as the alert channel for the triad row — the dashboard treats the eight metrics symmetrically.

## Where this hands off

- `dashboard.py` is the aggregation core of exercise-04's `dashboard.md`. In exercise-04 the per-item scores are produced by real RAGAS runs (`NoiseSensitivity`, `ContextUtilization`, `ResponseCompleteness`) plus a mod-104-style refusal-quality rubric; the aggregation / floor / alert logic you built here is unchanged.
- The `FLOORS` dict in `run.py` is the shape exercise-04's `alerting-policy.md` names per metric per slice; this lab gives you one defensible setting for the canned fixture, and exercise-04's self-consistency-variance stretch goal calibrates it against the actual metric noise of your real judge.
- The `regression-report.md` shape from step 10 is the shape mod-107's online-eval loop uses to page; the `app.rag.*` attributes on the mod-102 trace (chapter 04) are what populate each line of the report automatically.
- Lab-02's attribution decision tree is the natural next step from the "attribution: diagnostic-only" line in this report — once a diagnostic fires, the chapter-02 decision tree runs *on the affected slice* to attribute the underlying cause to retrieval or generation.
