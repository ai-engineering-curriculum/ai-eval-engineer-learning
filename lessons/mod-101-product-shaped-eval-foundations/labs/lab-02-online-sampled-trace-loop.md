# lab-02: Simulate the Online Sampled-Trace Loop

**Estimated duration:** 45–60 minutes
**Pairs with:** chapter 03 ("Offline, online, and deferred") and chapter 06 section 4 ("Online sampled-trace eval") of this module.

## Objective

Chapter 03 names three things offline eval cannot do: cover the live distribution, detect drift, or measure cost and latency under real load. All three live in the *online sampled-trace loop* — sample a share of production traces, score them, and watch an SLI against a watch-only or page-worthy SLO.

In this lab you simulate that loop end to end on a synthetic trace stream:

- A `traces.jsonl` representing a day of sampled production traffic with a cohort label on each row.
- A **stratified sampling policy** with a per-family cap (chapter 03 explicitly warns against uniform-random sampling).
- A deterministic **mock judge** that scores each sampled trace 1–5 on a single rubric.
- A runner that reports the rolling-window SLI, compares it to a watch-only SLO, and fires a page threshold when the rolling SLI drops far enough.
- A second run against a **drifted** trace file so you can watch the SLI move and the alert fire.

This lab does not duplicate exercise-02: that exercise places criteria into offline / online / deferred buckets and designs the sampling budget on paper. This lab takes the chapter-03 online-loop shape and runs it.

## Prerequisites

- Chapter 03 read; chapter 06 section 4 skimmed.
- Python 3.10+ (`python3 --version`).
- Lab-01 is not required but recommended — the file layout and the `plan.yaml` shape are deliberately parallel.
- A working directory you can write files into. No internet or paid API calls.

## Layout you will build

```
mod-101-lab-02/
  traces.jsonl
  traces_drifted.jsonl
  sampling.py
  mock_judge.py
  online_eval.py
  online_plan.yaml
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-101-lab-02 && cd mod-101-lab-02
```

### Step 2 — Generate a synthetic `traces.jsonl`

The surface stays the Meeting Summariser from chapter 02's running example. Each trace represents one summary that was served to a user and recorded for later scoring — the raw stream the sampling policy will draw from.

Create `generate_traces.py`:

```python
# generate_traces.py
"""Deterministic generator for synthetic traces.

Three cohorts stand in for the sorts of slices a real stratified
sampler cares about (chapter 03: 'stratify by cohort, by feature flag,
by traffic source'). Each trace carries a hidden 'true_quality' the
mock judge will later turn into a 1-5 score."""

import json
from pathlib import Path

COHORTS = {
    "enterprise-eng": 500,
    "smb-sales": 300,
    "support-triage": 200,
}

QUALITY_BY_COHORT = {
    "enterprise-eng": [5] * 92 + [3] * 5 + [1] * 3,
    "smb-sales": [5] * 94 + [3] * 4 + [1] * 2,
    "support-triage": [5] * 88 + [3] * 8 + [1] * 4,
}


def main(out_path: str, drifted: bool = False) -> None:
    rows: list[dict] = []
    trace_id = 0
    for cohort, n in COHORTS.items():
        pattern = QUALITY_BY_COHORT[cohort]
        if drifted and cohort == "support-triage":
            # Simulate a regression that affects only one cohort — the
            # chapter 03 case for why you stratify instead of sampling
            # uniformly.
            pattern = [5] * 60 + [3] * 20 + [1] * 20
        for i in range(n):
            trace_id += 1
            rows.append(
                {
                    "trace_id": f"t-{trace_id:04d}",
                    "cohort": cohort,
                    "true_quality": pattern[i % len(pattern)],
                }
            )
    Path(out_path).write_text("\n".join(json.dumps(r) for r in rows) + "\n")


if __name__ == "__main__":
    main("traces.jsonl", drifted=False)
    main("traces_drifted.jsonl", drifted=True)
    print("wrote traces.jsonl and traces_drifted.jsonl")
```

Run it:

```bash
python3 generate_traces.py
wc -l traces.jsonl traces_drifted.jsonl
```

**Expected:** both files have exactly 1000 lines. The `true_quality` field is the ground truth the mock judge in step 4 will read; in a real loop the judge would read the output text itself.

### Step 3 — Stratified sampling policy (`sampling.py`)

Chapter 03's common trap list: *"Sampling the online loop uniformly at random. Random sampling misses the tail. Stratify by cohort, by feature flag, by traffic source."* Implement a sampler that honours a global rate and a per-cohort cap so the smaller cohorts do not get swamped by the dominant one.

```python
# sampling.py
"""Stratified sampling with a per-cohort cap.

The global rate is the fraction of all traces we attempt to score.
The per-cohort cap is the maximum number of sampled traces per cohort
in a single run — it prevents the biggest cohort from eating the
judge budget while a small cohort goes unseen."""

from collections import defaultdict
from typing import Iterable


def stratified_sample(
    traces: Iterable[dict], rate: float, per_cohort_cap: int
) -> list[dict]:
    by_cohort: dict[str, list[dict]] = defaultdict(list)
    for t in traces:
        by_cohort[t["cohort"]].append(t)

    sampled: list[dict] = []
    for cohort, cohort_traces in by_cohort.items():
        step = max(1, int(round(1 / rate)))
        picked = cohort_traces[::step][:per_cohort_cap]
        sampled.extend(picked)
    return sampled
```

Deterministic by design: the lab is reproducible. In production you would use a random stream, but the policy shape is the same.

### Step 4 — Mock judge (`mock_judge.py`)

The judge stands in for an LLM-as-judge call. It returns the `true_quality` field verbatim. Chapter 03 lists "judge-vs-human calibration methodology" as deferred to the peer `model-evaluation-engineer`; this lab does *not* calibrate — the mock judge is explicitly a stand-in.

```python
# mock_judge.py
"""Deterministic mock judge.

Returns the trace's hidden true_quality. Chapter 03 lists judge-vs-
human calibration methodology (kappa, Spearman, bootstrap CIs at
methodology depth) as deferred to the peer model-evaluation-engineer;
the mock judge here does not calibrate."""


def score(trace: dict) -> int:
    return trace["true_quality"]
```

### Step 5 — Pre-register the watch-only SLO (`online_plan.yaml`)

Chapter 06 section 4 says the plan must name: sampling policy, which SLIs run online, the window, the cadence, the judge stack, and page thresholds. Encode that:

```yaml
# online_plan.yaml
surface: meeting-summariser
version: 0.1
sampling:
  rate: 0.10
  per_cohort_cap: 50
  stratify_by: cohort
window: one-run
sli:
  id: O1
  family: task-success
  name: share_judge_score_ge_4
  judge: mock_judge
  threshold: 4
slo:
  target: 0.92
  gate: watch-only
alerting:
  page_threshold: 0.85
```

Two important fields:

- `gate: watch-only` — chapter 03 is emphatic: online SLIs are lagging, so do not gate a merge on them. They are watch-only or canary targets.
- `page_threshold: 0.85` — a floor below the SLO at which someone is paged. In a real plan this is tied to the on-call rotation listed in chapter 06 section 4.

### Step 6 — The online loop (`online_eval.py`)

Reads the plan, loads the trace file, applies the sampler, runs the judge, computes the SLI, and reports against both the SLO (watch-only) and the page threshold.

```python
# online_eval.py
"""Simulated online sampled-trace loop.

Reads online_plan.yaml, loads a trace file, applies the stratified
sampler, runs the mock judge, computes the SLI, reports against the
watch-only SLO, and prints an alert state when the SLI drops below
the page threshold."""

import argparse
import json
from collections import defaultdict
from pathlib import Path

import yaml

import mock_judge
from sampling import stratified_sample


def load_traces(path: Path) -> list[dict]:
    with path.open() as fh:
        return [json.loads(line) for line in fh if line.strip()]


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--plan", default="online_plan.yaml")
    parser.add_argument("--traces", default="traces.jsonl")
    args = parser.parse_args()

    plan = yaml.safe_load(Path(args.plan).read_text())
    traces = load_traces(Path(args.traces))

    sampled = stratified_sample(
        traces,
        rate=plan["sampling"]["rate"],
        per_cohort_cap=plan["sampling"]["per_cohort_cap"],
    )

    threshold = plan["sli"]["threshold"]
    per_cohort_hits: dict[str, int] = defaultdict(int)
    per_cohort_total: dict[str, int] = defaultdict(int)
    hits = 0
    for t in sampled:
        score = mock_judge.score(t)
        per_cohort_total[t["cohort"]] += 1
        if score >= threshold:
            hits += 1
            per_cohort_hits[t["cohort"]] += 1

    sli = hits / len(sampled) if sampled else 0.0
    per_cohort_sli = {
        c: per_cohort_hits[c] / per_cohort_total[c] for c in per_cohort_total
    }

    report = {
        "surface": plan["surface"],
        "sampled_count": len(sampled),
        "sli": round(sli, 4),
        "slo_target": plan["slo"]["target"],
        "slo_gate": plan["slo"]["gate"],
        "page_threshold": plan["alerting"]["page_threshold"],
        "slo_met": sli >= plan["slo"]["target"],
        "paging": sli < plan["alerting"]["page_threshold"],
        "per_cohort_sli": {c: round(v, 4) for c, v in per_cohort_sli.items()},
    }
    print(json.dumps(report, indent=2))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Install and run:

```bash
pip install pyyaml
python3 online_eval.py
```

**Expected output (healthy run):**

- `sampled_count` is around 100 total (10% of 1000, capped at 50 per cohort → 50 + 30 + 20 given the cohort sizes).
- `sli` is high — well over `0.92` — because the synthetic cohort patterns were constructed to be mostly ≥4.
- `slo_met: true`, `paging: false`.
- `per_cohort_sli` shows all three cohorts near their healthy rates.

### Step 7 — Rerun against the drifted traces to watch the SLI move

The point of an online loop is detecting what offline cannot. Point the runner at `traces_drifted.jsonl`:

```bash
python3 online_eval.py --traces traces_drifted.jsonl
```

**Expected output (drifted run):**

- The overall `sli` drops. How far depends on the sampling arithmetic; the `support-triage` cohort's quality was degraded by construction.
- `per_cohort_sli["support-triage"]` drops noticeably compared with the healthy run — chapter 03's "stratify so the tail does not disappear in the aggregate" doing its job.
- Depending on the sample draw, `slo_met` may be `false` and `paging` may be `true`.

The difference between the two runs is exactly the signal a watch-only SLI is designed to carry. Nothing about this signal could come from the offline harness in lab-01, because the offline fixtures are fixed — chapter 03's "by construction, fixtures do not shift."

### Step 8 — Confirm the chapter-03 placement properties on this loop

Match the harness behaviour you just observed back onto chapter 03's columns. You should be able to point at concrete lines of code for each row:

| Chapter 03 claim | Where it shows up in this lab |
|---|---|
| Online owns live-distribution SLIs. | `sli = hits / len(sampled)` in `online_eval.py`. |
| Online owns drift detection. | The step-6 vs step-7 delta on `per_cohort_sli`. |
| Sampling policy is part of the plan. | `sampling:` block in `online_plan.yaml`. |
| Stratify, don't uniform-random. | `stratified_sample` in `sampling.py`. |
| Online SLIs do not gate merges. | `gate: watch-only` in the plan; `online_eval.py` always exits 0. |
| Judge-vs-human calibration is deferred. | Docstring of `mock_judge.py` names the peer owner. |

### Step 9 — Checkpoint

You are done with this lab when every statement below is true.

- [ ] `python3 online_eval.py` prints a report with `sli`, `per_cohort_sli`, `slo_met`, and `paging` fields.
- [ ] The healthy trace file produces `slo_met: true` and `paging: false`.
- [ ] The drifted trace file produces a visibly lower `per_cohort_sli["support-triage"]` compared with the healthy run.
- [ ] `online_eval.py` always exits 0, regardless of SLI value — online SLIs do not gate merges (chapter 03 and the `gate: watch-only` field in the plan).
- [ ] You can explain, in one sentence each, (a) why stratified sampling matters and (b) why the judge in this lab does not discharge the `model-evaluation-engineer` deferral.

## Where this hands off

- The `online_plan.yaml` file is the chapter 06 section 4 artefact in machine-readable form. Together with the `plan.yaml` from lab-01 it is the "offline + online" halves of the chapter 03 placement decision made concrete.
- `mod-107` (online eval and regression) replaces the mock judge with a real LLM-as-judge, adds canary and shadow methodology, and formalises the page thresholds into an on-call wiring.
- `mod-104` (LLM-as-judge in product) replaces `mock_judge.py` with a judge whose bias controls are documented.
- `mod-110` (eval-data-platform slice) ingests the reports produced here for history and drift trend-lines — the per-cohort SLI columns are specifically what a drift dashboard renders.
