# lab-02: Walk the Chapter-02 Attribution Decision Tree on Four Canned Regressions

**Estimated duration:** 45–60 minutes
**Pairs with:** chapter 02 of this module.

## Objective

Make the chapter-02 attribution decision tree mechanical. You build a tiny `attribute.py` that takes two score bundles (baseline vs candidate) — retrieval-only, generator replay, retriever replay, and joint triad — and prints one of four verdicts: `retrieval | generation | coupling | joint`. Then you walk four canned regression fixtures through it:

1. A **retrieval-only** regression (recall@5 moves; generator replay is flat).
2. A **generation-only** regression (generator replay moves; retrieval-only is flat).
3. A **coupling** regression (both flat individually; joint triad moves).
4. A **joint / coincident** regression (both moved independently in the same release window).

Each scenario's input is pre-canned so the attribution logic can be read off without running a retriever or a judge. By the end you can name, for any triad-regression dashboard shape, which branch of the tree the evidence lands on and what the chapter-02 handoff shape is — before you touch the code in exercise-02.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-02 asks you to build the full attribution machinery against your real surface's retriever and generator — `retrieval_metrics.py`, `generator_replay.py`, `retriever_replay.py`, and `attribute.py`, with two seeded regressions on real model calls and a 15-minute runbook.
- This lab builds just the **decision-tree logic** of `attribute.py` — the branching that reads the four score bundles and names the verdict — and runs it against canned fixtures. The I/O shape is identical; the retriever / generator / judge calls are replaced with pre-canned CSVs so the logic is testable without any model API.
- Lab-01 computed the retrieval-only metrics themselves (recall@k / MRR / hit rate) by hand; this lab assumes you have those numbers and focuses on what to do with them.

## Prerequisites

- Chapter 02 read.
- Lab-01 completed (you have the retrieval-only metrics in your head) — optional but recommended.
- Python 3.10+ on `$PATH`. Stdlib only.
- A working directory you can write files into. **No internet, no model API keys** — the fixtures are CSV, the decision tree is ~30 lines of if/elif.

## Layout you will build

```
mod-105-lab-02/
  fixtures/
    baseline/
      retrieval.csv        # recall@5 / MRR / hit_rate@5 per slice
      generator_replay.csv # faithfulness / answer_relevancy per slice (frozen context)
      retriever_replay.csv # faithfulness / answer_relevancy per slice (frozen generator)
      joint_triad.csv      # faithfulness / answer_relevancy / ctx_precision / ctx_recall per slice
    scenario_a_retrieval/
      *.csv                # same four files with retrieval-only perturbation
    scenario_b_generation/
      *.csv                # same four files with generation-only perturbation
    scenario_c_coupling/
      *.csv                # same four files with coupling perturbation
    scenario_d_joint/
      *.csv                # same four files with both perturbed independently
  attribute.py             # the decision tree
  run.py                   # CLI driver
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-105-lab-02/fixtures/baseline && cd mod-105-lab-02
for s in scenario_a_retrieval scenario_b_generation scenario_c_coupling scenario_d_joint; do
  mkdir -p "fixtures/$s"
done
ls fixtures/
```

**Expected:** five directories — `baseline/`, `scenario_a_retrieval/`, `scenario_b_generation/`, `scenario_c_coupling/`, `scenario_d_joint/`.

### Step 2 — Author the baseline score bundle

Four CSVs — one per column of chapter 02's attribution evidence — reporting per-slice means on the four slices from chapter 03 (`in_scope_single_chunk`, `in_scope_multi_chunk`, `distractor_heavy`, `out_of_scope`). The numbers are illustrative; they preserve chapter 02's shape so the decision tree's branches have unambiguous signal.

Create `fixtures/baseline/retrieval.csv`:

```csv
slice,n,recall_at_5,mrr,hit_rate_at_5
in_scope_single_chunk,40,1.00,1.00,1.00
in_scope_multi_chunk,40,0.79,1.00,1.00
distractor_heavy,40,0.85,0.95,1.00
out_of_scope,30,n/a,n/a,n/a
```

Create `fixtures/baseline/generator_replay.csv` (chapter 02's fixed-context replay — the retrieval is frozen, only the generator varies):

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.92,0.90
in_scope_multi_chunk,40,0.88,0.89
distractor_heavy,40,0.86,0.88
out_of_scope,30,n/a,n/a
```

Create `fixtures/baseline/retriever_replay.csv` (generator frozen, only the retriever varies — in baseline, identical to joint):

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.92,0.90
in_scope_multi_chunk,40,0.88,0.89
distractor_heavy,40,0.86,0.88
out_of_scope,30,n/a,n/a
```

Create `fixtures/baseline/joint_triad.csv` (the full-pipeline triad — what moves on the user-visible dashboard):

```csv
slice,n,faithfulness,answer_relevancy,context_precision,context_recall
in_scope_single_chunk,40,0.92,0.90,0.95,1.00
in_scope_multi_chunk,40,0.88,0.89,0.82,0.79
distractor_heavy,40,0.86,0.88,0.72,0.80
out_of_scope,30,n/a,n/a,n/a,n/a
```

### Step 3 — Author the four scenarios

Each scenario is a copy of the baseline with one targeted perturbation. The perturbation lives *only* in the bundles the tree reads for that branch — so the decision-tree logic has an unambiguous signal.

**Scenario A — retrieval-only regression.** `recall@5` on the multi-chunk slice drops 15 points; the frozen-context generator replay is flat (the generator never saw the broken retrieval); the joint triad drops because the live retrieval is broken.

Create `fixtures/scenario_a_retrieval/retrieval.csv`:

```csv
slice,n,recall_at_5,mrr,hit_rate_at_5
in_scope_single_chunk,40,1.00,1.00,1.00
in_scope_multi_chunk,40,0.64,0.88,0.95
distractor_heavy,40,0.85,0.95,1.00
out_of_scope,30,n/a,n/a,n/a
```

Create `fixtures/scenario_a_retrieval/generator_replay.csv` (identical to baseline — frozen context, no change):

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.92,0.90
in_scope_multi_chunk,40,0.88,0.89
distractor_heavy,40,0.86,0.88
out_of_scope,30,n/a,n/a
```

Create `fixtures/scenario_a_retrieval/retriever_replay.csv` (generator frozen, retriever broken — faithfulness drops because retrieval lost chunks):

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.92,0.90
in_scope_multi_chunk,40,0.78,0.85
distractor_heavy,40,0.86,0.88
out_of_scope,30,n/a,n/a
```

Create `fixtures/scenario_a_retrieval/joint_triad.csv`:

```csv
slice,n,faithfulness,answer_relevancy,context_precision,context_recall
in_scope_single_chunk,40,0.92,0.90,0.95,1.00
in_scope_multi_chunk,40,0.78,0.85,0.70,0.64
distractor_heavy,40,0.86,0.88,0.72,0.80
out_of_scope,30,n/a,n/a,n/a,n/a
```

**Scenario B — generation-only regression.** Retrieval-only metrics flat; the frozen-context generator replay drops (the new generator now paraphrases in a way the faithfulness judge marks unsupported — chapter-02's worked attribution); joint triad drops.

Create `fixtures/scenario_b_generation/retrieval.csv` (identical to baseline):

```csv
slice,n,recall_at_5,mrr,hit_rate_at_5
in_scope_single_chunk,40,1.00,1.00,1.00
in_scope_multi_chunk,40,0.79,1.00,1.00
distractor_heavy,40,0.85,0.95,1.00
out_of_scope,30,n/a,n/a,n/a
```

Create `fixtures/scenario_b_generation/generator_replay.csv` (frozen context + new generator — faithfulness drops):

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.85,0.89
in_scope_multi_chunk,40,0.80,0.87
distractor_heavy,40,0.78,0.86
out_of_scope,30,n/a,n/a
```

Create `fixtures/scenario_b_generation/retriever_replay.csv` (frozen generator + baseline retriever — identical to baseline):

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.92,0.90
in_scope_multi_chunk,40,0.88,0.89
distractor_heavy,40,0.86,0.88
out_of_scope,30,n/a,n/a
```

Create `fixtures/scenario_b_generation/joint_triad.csv`:

```csv
slice,n,faithfulness,answer_relevancy,context_precision,context_recall
in_scope_single_chunk,40,0.85,0.89,0.95,1.00
in_scope_multi_chunk,40,0.80,0.87,0.82,0.79
distractor_heavy,40,0.78,0.86,0.72,0.80
out_of_scope,30,n/a,n/a,n/a,n/a
```

**Scenario C — coupling regression.** Both retrieval-only *and* generator replay are within noise floor; the joint triad moves because the retriever and generator interact (chapter-02's "new retrieval, old generator prompt" case — the retriever now surfaces chunks the prompt template's instructions don't handle well).

Create `fixtures/scenario_c_coupling/retrieval.csv` (within noise of baseline):

```csv
slice,n,recall_at_5,mrr,hit_rate_at_5
in_scope_single_chunk,40,1.00,1.00,1.00
in_scope_multi_chunk,40,0.78,1.00,1.00
distractor_heavy,40,0.84,0.94,1.00
out_of_scope,30,n/a,n/a,n/a
```

Create `fixtures/scenario_c_coupling/generator_replay.csv` (within noise of baseline):

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.92,0.90
in_scope_multi_chunk,40,0.87,0.89
distractor_heavy,40,0.86,0.88
out_of_scope,30,n/a,n/a
```

Create `fixtures/scenario_c_coupling/retriever_replay.csv` (within noise of baseline):

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.92,0.90
in_scope_multi_chunk,40,0.87,0.89
distractor_heavy,40,0.86,0.88
out_of_scope,30,n/a,n/a
```

Create `fixtures/scenario_c_coupling/joint_triad.csv` (joint drops because the new retriever's chunks hit the old prompt's blind spots):

```csv
slice,n,faithfulness,answer_relevancy,context_precision,context_recall
in_scope_single_chunk,40,0.92,0.90,0.95,1.00
in_scope_multi_chunk,40,0.79,0.86,0.80,0.78
distractor_heavy,40,0.75,0.84,0.70,0.80
out_of_scope,30,n/a,n/a,n/a,n/a
```

**Scenario D — joint / coincident regression.** Retrieval-only *and* generator replay both moved independently — a coincident release.

Create `fixtures/scenario_d_joint/retrieval.csv`:

```csv
slice,n,recall_at_5,mrr,hit_rate_at_5
in_scope_single_chunk,40,1.00,1.00,1.00
in_scope_multi_chunk,40,0.64,0.88,0.95
distractor_heavy,40,0.85,0.95,1.00
out_of_scope,30,n/a,n/a,n/a
```

Create `fixtures/scenario_d_joint/generator_replay.csv`:

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.85,0.89
in_scope_multi_chunk,40,0.80,0.87
distractor_heavy,40,0.78,0.86
out_of_scope,30,n/a,n/a
```

Create `fixtures/scenario_d_joint/retriever_replay.csv`:

```csv
slice,n,faithfulness,answer_relevancy
in_scope_single_chunk,40,0.92,0.90
in_scope_multi_chunk,40,0.78,0.85
distractor_heavy,40,0.86,0.88
out_of_scope,30,n/a,n/a
```

Create `fixtures/scenario_d_joint/joint_triad.csv`:

```csv
slice,n,faithfulness,answer_relevancy,context_precision,context_recall
in_scope_single_chunk,40,0.85,0.89,0.95,1.00
in_scope_multi_chunk,40,0.70,0.83,0.68,0.64
distractor_heavy,40,0.78,0.86,0.72,0.80
out_of_scope,30,n/a,n/a,n/a,n/a
```

### Step 4 — Write `attribute.py`

The decision tree, in code. The threshold `delta_floor` is the "material regression" cutoff per chapter 02 — anything inside `±delta_floor` on a slice counts as "flat"; anything outside counts as a move. 0.03 (three absolute points) is a defensible floor on the per-slice sample sizes in these fixtures; chapter 02 is explicit that the threshold is a surface-specific knob (see exercise-02's stretch-goal bootstrap CI).

```python
# attribute.py
"""Chapter-02 attribution decision tree.

Takes two score bundles (baseline vs candidate) — each a directory
containing retrieval.csv, generator_replay.csv, retriever_replay.csv,
joint_triad.csv — and emits one of four verdicts:

  retrieval  — retrieval-only metrics moved; replay flat
  generation — retrieval-only flat; generator replay moved
  coupling   — both flat individually; joint moved
  joint      — both moved independently (coincident release)

The tree does not resolve *which* retriever change or *which* generator
change caused the regression — that is the job of the downstream bisect
(the chapter-02 fix step, which lives in exercise-02 against your real
diff). This CLI tells you which side to escalate to."""

from __future__ import annotations

import csv
import json
from pathlib import Path


# --- CSV loader ------------------------------------------------------------

def load_csv(path: Path) -> list[dict]:
    rows = []
    with path.open() as f:
        for row in csv.DictReader(f):
            parsed = {}
            for k, v in row.items():
                if k in ("slice", "n"):
                    parsed[k] = int(v) if k == "n" else v
                else:
                    parsed[k] = None if v == "n/a" else float(v)
            rows.append(parsed)
    return rows


# --- delta on one metric column, per in-scope slice ------------------------

def slice_deltas(before: list[dict], after: list[dict], metric: str) -> dict:
    out = {}
    before_by_slice = {r["slice"]: r for r in before}
    for r in after:
        b = before_by_slice[r["slice"]]
        if b[metric] is None or r[metric] is None:
            out[r["slice"]] = None
            continue
        out[r["slice"]] = round(r[metric] - b[metric], 4)
    return out


# --- "material" delta test --------------------------------------------------

def any_material(delta_map: dict, delta_floor: float) -> bool:
    return any(d is not None and abs(d) >= delta_floor for d in delta_map.values())


# --- the decision tree ------------------------------------------------------

def attribute(baseline_dir: Path, candidate_dir: Path, delta_floor: float = 0.03) -> dict:
    # Load both score bundles.
    retr_b = load_csv(baseline_dir / "retrieval.csv")
    retr_c = load_csv(candidate_dir / "retrieval.csv")
    gen_b  = load_csv(baseline_dir / "generator_replay.csv")
    gen_c  = load_csv(candidate_dir / "generator_replay.csv")
    ret_b  = load_csv(baseline_dir / "retriever_replay.csv")
    ret_c  = load_csv(candidate_dir / "retriever_replay.csv")
    joint_b = load_csv(baseline_dir / "joint_triad.csv")
    joint_c = load_csv(candidate_dir / "joint_triad.csv")

    # Retrieval-only signal: any of recall@5 / MRR / hit_rate@5 materially moved.
    retr_deltas = {m: slice_deltas(retr_b, retr_c, m) for m in ("recall_at_5", "mrr", "hit_rate_at_5")}
    retrieval_moved = any(any_material(d, delta_floor) for d in retr_deltas.values())

    # Generator-replay signal: faithfulness or answer_relevancy moved on frozen context.
    gen_deltas = {m: slice_deltas(gen_b, gen_c, m) for m in ("faithfulness", "answer_relevancy")}
    generator_moved = any(any_material(d, delta_floor) for d in gen_deltas.values())

    # Joint-triad signal (used to distinguish coupling from "nothing moved").
    joint_deltas = {
        m: slice_deltas(joint_b, joint_c, m)
        for m in ("faithfulness", "answer_relevancy", "context_precision", "context_recall")
    }
    joint_moved = any(any_material(d, delta_floor) for d in joint_deltas.values())

    # Retriever replay (used as a sanity check on the retrieval side).
    ret_deltas = {m: slice_deltas(ret_b, ret_c, m) for m in ("faithfulness", "answer_relevancy")}

    # The chapter-02 decision tree.
    if retrieval_moved and generator_moved:
        verdict = "joint"
    elif retrieval_moved and not generator_moved:
        verdict = "retrieval"
    elif generator_moved and not retrieval_moved:
        verdict = "generation"
    elif joint_moved:
        verdict = "coupling"
    else:
        verdict = "no_regression"

    return {
        "verdict": verdict,
        "delta_floor": delta_floor,
        "retrieval_moved": retrieval_moved,
        "generator_moved": generator_moved,
        "joint_moved": joint_moved,
        "retrieval_deltas": retr_deltas,
        "generator_replay_deltas": gen_deltas,
        "retriever_replay_deltas": ret_deltas,
        "joint_triad_deltas": joint_deltas,
    }


if __name__ == "__main__":  # pragma: no cover
    import sys
    print(json.dumps(attribute(Path(sys.argv[1]), Path(sys.argv[2])), indent=2))
```

### Step 5 — Write the one-screen driver (`run.py`)

```python
# run.py
"""Prints the chapter-02 attribution verdict for one scenario, one screen."""

import sys
from pathlib import Path

from attribute import attribute


HANDOFF = {
    "retrieval":  "Escalate to the rag-engineer peer with the slice-labelled report (chapter 05).",
    "generation": "Own on this side — bisect the generator diff (prompt / model / tool config).",
    "coupling":   "Both owners on the hook — drive from whoever released last, else joint review.",
    "joint":      "Coincident release — each side handles its own; agree release ordering to avoid re-testing.",
    "no_regression": "No material regression on this delta_floor.",
}


def main() -> int:
    baseline = Path(sys.argv[1])
    candidate = Path(sys.argv[2])
    out = attribute(baseline, candidate)

    print(f"## Scenario: {candidate.name}")
    print()
    print(f"VERDICT: {out['verdict'].upper()}")
    print(f"Handoff: {HANDOFF[out['verdict']]}")
    print(f"delta_floor = {out['delta_floor']}")
    print()
    print("Retrieval-only deltas (recall@5 / MRR / hit_rate@5), per slice:")
    for m, d in out["retrieval_deltas"].items():
        print(f"  {m:14s}: {d}")
    print()
    print("Generator replay deltas (faithfulness / answer_relevancy), per slice [frozen context]:")
    for m, d in out["generator_replay_deltas"].items():
        print(f"  {m:14s}: {d}")
    print()
    print("Retriever replay deltas (faithfulness / answer_relevancy), per slice [frozen generator]:")
    for m, d in out["retriever_replay_deltas"].items():
        print(f"  {m:14s}: {d}")
    print()
    print("Joint triad deltas, per slice [live pipeline]:")
    for m, d in out["joint_triad_deltas"].items():
        print(f"  {m:14s}: {d}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### Step 6 — Walk scenario A (retrieval-only)

```bash
python3 run.py fixtures/baseline fixtures/scenario_a_retrieval
```

**Expected:**

- `VERDICT: RETRIEVAL`.
- `retrieval_deltas["recall_at_5"]["in_scope_multi_chunk"] ≈ -0.15` — the perturbation.
- `generator_replay_deltas["faithfulness"]` is `{...: 0.0, ...: 0.0, ...: 0.0}` on in-scope slices — the frozen-context generator replay is flat by construction.
- `joint_triad_deltas["faithfulness"]["in_scope_multi_chunk"] ≈ -0.10` — the joint moved because live retrieval broke.
- Handoff line: "Escalate to the rag-engineer peer with the slice-labelled report (chapter 05)."

Why: chapter 02's tree branch 1 — `retrieval-only metrics moved` — resolves to the retrieval owner. The frozen-context generator replay being flat is the *sanity check* that confirms the generator is not at fault; it would be a mistake to page the generator team on this.

### Step 7 — Walk scenario B (generation-only)

```bash
python3 run.py fixtures/baseline fixtures/scenario_b_generation
```

**Expected:**

- `VERDICT: GENERATION`.
- `retrieval_deltas` are all 0.0 on the in-scope slices (retrieval unchanged).
- `generator_replay_deltas["faithfulness"]["in_scope_single_chunk"] ≈ -0.07`; multi-chunk ≈ -0.08; distractor_heavy ≈ -0.08 — the frozen-context replay sees the generator regress even without retrieval moving.
- `joint_triad_deltas["faithfulness"]` reflects the same shape.
- Handoff line: "Own on this side — bisect the generator diff."

Why: chapter 02's tree branch 2. The replay running on the *same* retrieval the baseline saw is the chapter-02 fixed-context replay — any delta there is a generation regression by construction. This is the chapter's worked-attribution scenario on paper.

### Step 8 — Walk scenario C (coupling)

```bash
python3 run.py fixtures/baseline fixtures/scenario_c_coupling
```

**Expected:**

- `VERDICT: COUPLING`.
- `retrieval_deltas["recall_at_5"]["in_scope_multi_chunk"] ≈ -0.01` — within floor.
- `generator_replay_deltas["faithfulness"]["in_scope_multi_chunk"] ≈ -0.01` — within floor.
- `joint_triad_deltas["faithfulness"]["distractor_heavy"] ≈ -0.11` — joint moved.
- Handoff line: "Both owners on the hook — drive from whoever released last, else joint review."

Why: chapter 02 is explicit that when both isolated metrics are flat *and* the joint still shows a regression, the retriever and generator interact. The chapter names two sub-cases — new retrieval meets old prompt (the generator is fine on fresh chunks but the prompt template's instructions don't handle their shape), or new generator meets old retrieval (paraphrases the judge's decomposer misreads). The CLI can't resolve the sub-case automatically, but the verdict tells the on-call who's on the hook.

### Step 9 — Walk scenario D (joint / coincident)

```bash
python3 run.py fixtures/baseline fixtures/scenario_d_joint
```

**Expected:**

- `VERDICT: JOINT`.
- `retrieval_deltas["recall_at_5"]["in_scope_multi_chunk"] ≈ -0.15`.
- `generator_replay_deltas["faithfulness"]` shows a drop across in-scope slices.
- `joint_triad_deltas` reflects the combined drop (larger than either side alone on the multi-chunk slice).
- Handoff line: "Coincident release — each side handles its own; agree release ordering to avoid re-testing."

Why: chapter 02's fourth outcome. Two independent regressions in the same release window. The tree does not try to merge them into one fix — it names both drivers so neither gets lost.

### Step 10 — Sweep the delta_floor

Re-run each scenario with a tighter and a looser floor:

```bash
python3 - <<'PY'
import json
from pathlib import Path
from attribute import attribute

for scen in ["scenario_a_retrieval", "scenario_b_generation",
             "scenario_c_coupling", "scenario_d_joint"]:
    for fl in (0.01, 0.03, 0.10):
        v = attribute(Path("fixtures/baseline"), Path(f"fixtures/{scen}"), delta_floor=fl)["verdict"]
        print(f"  floor={fl} scen={scen:28s} verdict={v}")
PY
```

**Expected shape:**

- At `floor = 0.01` (very tight), **scenario C** flips from `coupling` to `joint` — the within-noise ~0.01 residuals on retrieval and generator replay now count as material, so both flags fire together. This is the chapter-02 point about threshold choice: if the floor is tighter than your actual metric noise, routine jitter reads as a coincident regression.
- At `floor = 0.10` (loose), **scenario B** reads as `no_regression` because its ~7–8-point generator-replay drop falls under the floor, and **scenario D** flips from `joint` to `retrieval` because the retrieval drop (−0.15) still crosses the floor while the generator-replay drop (~0.08) no longer does. This is the chapter-02 point from the other direction: if the floor is looser than the real regression, you ship it — or you page only half of a coincident release.
- At `floor = 0.03` (the lab's default for these sample sizes), the four scenarios map cleanly to the four verdicts.

The right floor is a per-slice function of the sample size and the metric's noise floor (chapter-04 on diagnostic noise floor; exercise-04's stretch-goal on self-consistency variance; exercise-02's stretch-goal on bootstrap CIs). For this fixture, 0.03 is defensible; for a production surface, calibrate it against actual metric variance before reading any verdict as actionable.

### Step 11 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `python3 run.py fixtures/baseline fixtures/scenario_a_retrieval` prints `VERDICT: RETRIEVAL`, and the per-slice deltas show retrieval-only moved while the frozen-context generator replay is flat.
- [ ] Scenario B prints `VERDICT: GENERATION`, with retrieval deltas 0.0 and generator-replay deltas negative on in-scope slices.
- [ ] Scenario C prints `VERDICT: COUPLING`, with retrieval and generator-replay deltas both within the floor and joint-triad deltas outside it.
- [ ] Scenario D prints `VERDICT: JOINT`, with both retrieval and generator-replay deltas outside the floor.
- [ ] You can name, for each verdict, which part of the attribution evidence collapsed the tree to that branch — not "the verdict said so" but "retrieval-only moved while the frozen-context generator replay didn't".
- [ ] You can explain why the frozen-context generator replay being flat in scenario A is a *sanity check* that would be a mistake to skip — without it, a joint-triad drop on a retrieval regression could be misread as a generation issue.
- [ ] You can explain how the `delta_floor` choice changes which scenarios read as regressions, and you can point at the chapter-02 / chapter-04 references that calibrate the floor against the slice-sample noise.
- [ ] You can state which `app.rag.*` attribute on the mod-102 trace each of the four CSVs (retrieval / generator_replay / retriever_replay / joint_triad) would be reconstructed from in a production attribution run.

## Where this hands off

- `attribute.py` is the decision-tree core of exercise-02's `attribute.py`. In exercise-02 the four CSVs are replaced by live runs against your real retriever (for retrieval metrics), your pinned generator (for the retriever replay), and your frozen retrieval (for the generator replay); the branching logic you wrote here is unchanged.
- The handoff lines in `run.py` are the one-sentence versions of the chapter-05 handoff template — in a production run, the retrieval verdict triggers the full slice-labelled regression report the `rag-engineer` peer consumes.
- The `delta_floor` parameter is the per-slice material-regression threshold mod-106's CI gate and mod-107's online-eval alerting policy both read; this lab makes its sensitivity visible before production tuning.
- Lab-03 composes this decision-tree core with the chapter-04 diagnostic metrics so a diagnostic-only regression (noise sensitivity on the distractor_heavy slice, triad aggregate flat) pages even when none of the four branches above fire.
