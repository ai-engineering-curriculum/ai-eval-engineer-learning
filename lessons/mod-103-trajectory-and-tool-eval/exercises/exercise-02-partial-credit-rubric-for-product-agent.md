# exercise-02: Partial Credit Rubric For A Product Agent

**Estimated effort:** 3 hours

## Objective

Assemble the chapter-03 partial-credit rubric for a real product agent on top of the per-step and budget verdicts from exercise-01. The deliverable is a pre-registered rubric file plus a roll-up function that reads per-step, final-answer, groundedness, and budget inputs and emits the per-trajectory verdict JSON (scalar `trajectory_score`, `pass`, `axes`, `flagged_verdicts`, `floor_tripped`). The point is to make the attribution the rubric preserves visible — not to produce a smaller number.

By the end you will have the per-trajectory verdict shape that `mod-106` CI reads to gate merges and that `mod-107` online eval attaches back to the trace as an `EVALUATOR` span.

## Prerequisites

- Exercise-01 complete, with the per-step and budget scorers emitting the chapter-02 and chapter-04 JSON shapes.
- Chapter 03 of this module.
- A choice of final-answer scoring method (chapter 03 lists six). For this exercise pick two: one deterministic (exact-match or token-F1 on a normalised reply) and one stub (reference-based judge — stub it; mod-104 owns the depth).
- No new third-party dependencies required.

## Choose the task family and author the rubric file

Use the running `order_status_not_shipped_policy` task family or your own authorised surface. Author the rubric as a versioned YAML (or JSON) file:

```yaml
# support_agent/order_status_not_shipped_policy/rubric.v1.4.yaml
task_family: order_status_not_shipped_policy
rubric_version: v1.4
weights:
  name:          0.20
  args:          0.10
  order:         0.10
  answer:        0.20
  groundedness:  0.25
  budget:        0.15
floors:
  trip_on_verdict:
    - wrong_tool
    - unauthorised_tool
    - ordering_violation
    - fabricated_claim
pass_threshold: 0.80
methods:
  answer_scorer:        exact_match_normalised
  groundedness_scorer:  stub_v0           # mod-105 fills in
```

Three design rules the chapter emphasised:

- The weights sum to 1.0 (enforce it).
- Floors trip on a named verdict list, not on a threshold on a credit. A floored verdict fails the trajectory regardless of the weighted sum.
- Per-task-family files. Do not share one rubric across unrelated families.

## Requirements

Produce a directory `mod-103/exercise-02/` in your working repo containing:

1. **`rubric/schema.py`** — Pydantic (or dataclass) definitions for `RubricConfig`, `PerTrajectoryVerdict`, `AxisCredit`. The schema is the API this exercise exposes; keep it tight.
2. **`rubric/rollup.py`** — a function `roll_up(per_step: list, final_answer: dict, groundedness: dict, budget: dict, rubric: RubricConfig) -> PerTrajectoryVerdict` implementing chapter 03:
   - Credit-to-step attribution: worst-case per axis for `name_credit`, `args_credit`, `order_credit`.
   - Terminal-step attribution for `answer_credit` and `groundedness_credit`.
   - `budget_credit` passed through from exercise-01's budget scorer.
   - Weighted sum.
   - Floor check against `trip_on_verdict`. If tripped, `pass=false` regardless of the sum; `floor_tripped` names the first offending verdict.
   - Full `flagged_verdicts` list: union of chapter-02, chapter-03, and chapter-04 verdicts observed.
   - Preserve the per-axis `axes` vector in the output.
3. **`rubric/final_answer.py`** — two concrete scoring methods:
   - `exact_match_normalised(reply: str, reference: str) -> AxisCredit`: lowercase, strip whitespace, strip Markdown fences, string-compare. Return `{credit: 0.0 | 1.0, method: "exact_match_normalised"}`.
   - `token_f1(reply: str, reference: str) -> AxisCredit`: SQuAD-style token F1 with a note on when it is misleading.
   - A stub `reference_based_judge(reply: str, reference: str) -> AxisCredit` that raises `NotImplementedError("mod-104 owns judge configuration")`.
4. **`rubric/aggregate.py`** — a function `aggregate(verdicts: list[PerTrajectoryVerdict], slice_by: list[str]) -> dict` that computes per-slice `pass_rate`, `cost_p95`, `latency_p95`, `rate_of_flagged_verdict` across many trajectories. The slices default to `task_family`, `rubric_version`, and `cohort`.
5. **`tests/test_rollup.py`** — table-driven tests:
   - The chapter-01 bad trajectory (expect `pass=false`, `floor_tripped=wrong_tool`, `trajectory_score ≈ 0.455` within 0.05 tolerance, `flagged_verdicts` carrying both chapter-02 and chapter-04 entries).
   - A trajectory where the weighted sum is below `pass_threshold` with no floor tripped (expect `pass=false`, `floor_tripped=null`).
   - A trajectory where every axis is 1.0 (expect `pass=true`, `trajectory_score=1.0`).
   - A trajectory with `name_credit=0.9, args_credit=1.0, order_credit=1.0, answer_credit=1.0, groundedness_credit=1.0, budget_credit=1.0` — i.e. one small step problem but no floored verdict (expect `pass=true` because weighted sum is above threshold and no floor tripped).
6. **`README.md`** — 200–300 words describing the rubric file format, the roll-up semantics, and three things that would make the rubric wrong (post-hoc weight tuning, floor-free rubrics, one-rubric-across-families).

## Starter guidance

- Keep the rubric *file* separate from the rubric *code*. The file is a product artefact that lives alongside the eval plan; the code reads it.
- Enforce `sum(weights) == 1.0` on load; fail loudly on mismatches.
- The floor check runs *before* the weighted sum reporting. If a floor is tripped, still compute and return the weighted sum — a reviewer wants to see "the floor tripped, and additionally the sum was 0.45".
- Credit-to-step worst-case: for `name_credit`, treat each required step's name as a 0/1 variable and take the min. A step whose tool is not in the required set (an extra tool) does not lower `name_credit` by itself — it may contribute to `flagged_verdicts` as `extra_tool`, which is a separate floor concern.
- For `aggregate`, use `numpy.percentile(..., method='nearest')` or `statistics.quantiles` with a documented choice. Do not reinvent the percentile.
- Do **not** auto-tune weights. The weights come from the file; the code reads them.

## Acceptance criteria

You are done when:

- The rubric YAML loads, weights sum validates, and all five test cases pass.
- The roll-up function produces a `PerTrajectoryVerdict` with the fields exactly as chapter 03's example emits them.
- Running the full pipeline (`exercise-01` scorer → `exercise-02` rubric) on the chapter-01 bad trajectory produces `pass=false` and `floor_tripped=wrong_tool`.
- The aggregator emits per-slice numbers that match hand-computed values on a small (10–20 trajectory) fixture.
- The README names three failure modes the rubric is designed to prevent (judge-grading the whole trajectory, floors-free weighted sums, cross-family rubrics).
- `rubric_version` and `price_table_version` round-trip from input to output; a trajectory scored under `rubric_version=v1.4` carries `v1.4` in the verdict.
- Nothing in the roll-up calls a model.

## Stretch goals

- **Rubric diff tool.** Write `rubric/diff.py` that takes two rubric files and reports which weights changed and which floors were added or removed. Use it to compare `v1.4` to a hypothetical `v1.5` where `w_budget` moved from 0.15 to 0.25.
- **Historical re-score.** Given a sample of 50 historical trajectories scored under `v1.4`, re-score them under a candidate `v1.5` and emit a side-by-side pass-rate table. This is the artefact a rubric PR has to attach.
- **Per-cohort floor overrides.** Allow the rubric to specify stricter floors for a named cohort (e.g., EU locale, enterprise tier). Verify two trajectories identical except for cohort can trip different floors.
- **Rubric-vs-leaderboard note.** Write a 200-word note defending why your rubric differs from the τ-bench scoring shape your surface's PM asked about. Cite chapter 06.

## What this exercise does *not* cover

You are not implementing the judge (mod-104), you are not wrapping any of this in Inspect (exercise-03), you are not auditing public benchmarks (exercise-04), and you are not replaying anything (exercise-05). The groundedness scorer is a stub here; mod-105 fills it in. The judge is a stub here; mod-104 fills it in.
