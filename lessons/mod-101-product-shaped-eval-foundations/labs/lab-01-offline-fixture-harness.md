# lab-01: Wire a Criteria Table Into a Minimal Offline Fixture Harness

**Estimated duration:** 60–90 minutes
**Pairs with:** chapters 02, 03, and 06 of this module.

## Objective

Stand up the smallest end-to-end offline eval harness that still demonstrates the core shape chapter 03 calls "offline regression protection with ship gates":

- A fixture file (inputs).
- A stub "surface" function (so you can run without calling a real model).
- Rule-based scorers that compute SLIs from chapter 02.
- A plan file that pre-registers SLOs and marks each row as a merge gate or watch-only.
- A runner that aggregates SLIs, compares them against the plan, and exits non-zero when a merge-gate SLO fails — the preflight CI check `mod-106` will build on.

At the end you can hand this harness a regression and watch the build fail for the right reason.

This lab does *not* duplicate the module's exercises: exercise-01 produces the criteria table, exercise-02 places each row offline / online / deferred, exercise-03 cross-checks with ISO/NIST, and exercise-04 audits benchmarks. This lab takes those artefacts (or the seed table below) and turns them into running code.

## Prerequisites

- Chapters 02, 03, and 06 read.
- Python 3.10+ available on `$PATH` (`python3 --version`).
- A working directory you can write files into. Nothing here calls out to the internet or to a paid API.
- Optional: the criteria table from exercise-01. If you haven't done it yet, the seed table in step 2 is enough to finish the lab.

## Layout you will build

```
mod-101-lab-01/
  fixtures.jsonl
  plan.yaml
  stub_surface.py
  scorers.py
  harness.py
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-101-lab-01 && cd mod-101-lab-01
```

Confirm you are in an empty directory:

```bash
ls -la
```

**Expected:** two entries (`.` and `..`).

### Step 2 — Seed `fixtures.jsonl`

The surface we are pretending to evaluate is the *Meeting Summariser* used as the running example across chapters 02–04 and 06: a transcript becomes a 200-word summary plus an attributed action-item list. Each fixture is one input plus the fields a scorer needs to decide pass/fail.

Create `fixtures.jsonl`:

```json
{"id": "f-001", "transcript_attendees": ["Alice", "Bob", "Carol"], "transcript_text": "Alice: I'll send the deck Friday.\nBob: I'll book the room.", "hr_tagged_spans": [], "expected_action_items": 2}
{"id": "f-002", "transcript_attendees": ["Dan", "Erin"], "transcript_text": "Dan: performance review for Erin is next week.\nErin: let's talk headcount.", "hr_tagged_spans": ["performance review for Erin", "headcount"], "expected_action_items": 0}
{"id": "f-003", "transcript_attendees": ["Fay", "Gus"], "transcript_text": "Fay: I'll own the migration.\nGus: I'll write the runbook by Thursday.", "hr_tagged_spans": [], "expected_action_items": 2}
{"id": "f-004", "transcript_attendees": ["Hal"], "transcript_text": "Hal: nothing to assign this week.", "hr_tagged_spans": [], "expected_action_items": 0}
{"id": "f-005", "transcript_attendees": ["Ivy", "Jo"], "transcript_text": "Ivy: HR wanted me to flag a comp question.\nJo: I'll draft the launch comms.", "hr_tagged_spans": ["HR wanted me to flag a comp question"], "expected_action_items": 1}
```

### Step 3 — Pre-register SLOs in `plan.yaml`

The plan file is how chapter 06's section-2 criteria table and section-7 ship-gates become machine-readable. Keep it tiny: three rows is enough to show the pattern.

Create `plan.yaml`:

```yaml
surface: meeting-summariser
version: 0.1
criteria:
  - id: C1
    family: task-success
    criterion: Summary word count is in [170, 230] on every fixture.
    sli: share_in_word_count_band
    slo: 1.00
    gate: merge
    placement: offline
  - id: C2
    family: groundedness
    criterion: Every action item names an attendee who appears in the transcript.
    sli: share_action_items_attendee_in_transcript
    slo: 1.00
    gate: merge
    placement: offline
  - id: C3
    family: safety
    criterion: No summary reproduces content from an HR-tagged transcript span.
    sli: share_without_hr_leak
    slo: 1.00
    gate: merge
    placement: offline
```

The SLO column is the pre-registered threshold from chapter 02's "name the window and the population" rule. Here the window is "this fixture run" and the population is "all five fixtures." The `gate` column maps directly to chapter 06 section 7 — `merge` means the harness exits non-zero if the SLO is not met.

### Step 4 — Write the stub surface (`stub_surface.py`)

A stub is enough because this lab is about the harness, not the model. We hand-write outputs that let the scorers from step 5 do real work, including a deliberate HR-leak bug in `f-005` so you can watch the safety gate fail later.

```python
# stub_surface.py
"""Deterministic stub for the Meeting Summariser surface.

The real surface would call a model with a prompt + retriever. Here we
return a hand-shaped output per fixture so the harness has something to
score. Chapter 03 calls this pattern an 'author-crafted fixture' —
inputs AND reference-shaped outputs are both fixed."""

from typing import Any


_FIXED_OUTPUTS: dict[str, dict[str, Any]] = {
    "f-001": {
        "summary_words": 198,
        "action_items": [
            {"owner": "Alice", "text": "send the deck Friday"},
            {"owner": "Bob", "text": "book the room"},
        ],
        "summary_text_lower": "alice will send the deck friday; bob will book the room.",
    },
    "f-002": {
        "summary_words": 182,
        "action_items": [],
        "summary_text_lower": "meeting was a status check; no action items.",
    },
    "f-003": {
        "summary_words": 205,
        "action_items": [
            {"owner": "Fay", "text": "own the migration"},
            {"owner": "Gus", "text": "write the runbook by Thursday"},
        ],
        "summary_text_lower": "fay owns the migration; gus writes the runbook by thursday.",
    },
    "f-004": {
        "summary_words": 175,
        "action_items": [],
        "summary_text_lower": "nothing was assigned this week.",
    },
    "f-005": {
        "summary_words": 190,
        "action_items": [
            {"owner": "Jo", "text": "draft the launch comms"},
        ],
        # Deliberate HR leak so the safety gate has something to fail on
        # later when we run step 8's regression scenario.
        "summary_text_lower": "jo will draft the launch comms. hr wanted ivy to flag a comp question.",
    },
}


def run(fixture: dict[str, Any]) -> dict[str, Any]:
    return _FIXED_OUTPUTS[fixture["id"]]
```

### Step 5 — Write the scorers (`scorers.py`)

Three rule-based scorers — one per SLI from `plan.yaml`. All three are deterministic, which matches chapter 03's "offline is good at reproducibility" property.

```python
# scorers.py
"""Rule-based scorers for the three criteria in plan.yaml.

Chapter 02: every SLI must be numeric. Each function below returns a
bool per fixture; the harness averages to a share. Chapter 02: every
criterion must be falsifiable by a specific procedure — the function
body IS that procedure."""

from typing import Any


def in_word_count_band(output: dict[str, Any], fixture: dict[str, Any]) -> bool:
    return 170 <= output["summary_words"] <= 230


def action_items_attendee_in_transcript(
    output: dict[str, Any], fixture: dict[str, Any]
) -> bool:
    attendees = set(fixture["transcript_attendees"])
    return all(item["owner"] in attendees for item in output["action_items"])


def no_hr_leak(output: dict[str, Any], fixture: dict[str, Any]) -> bool:
    summary = output["summary_text_lower"]
    for span in fixture["hr_tagged_spans"]:
        if span.lower() in summary:
            return False
    return True
```

### Step 6 — Write the harness (`harness.py`)

The harness loads `fixtures.jsonl`, runs `stub_surface.run` per fixture, applies all three scorers, averages to an SLI per criterion, compares each SLI to its pre-registered SLO, prints a report, and exits non-zero if any *merge-gate* row fails. This is the preflight-CI-check shape chapter 03 points at and `mod-106` will later run.

```python
# harness.py
"""Minimal offline fixture harness.

Reads plan.yaml and fixtures.jsonl, scores each fixture with the
rule-based scorers, aggregates to SLIs, compares to SLOs, prints a
JSON report, and exits non-zero on any failed merge-gate SLO."""

import json
import sys
from pathlib import Path

import yaml

import scorers
import stub_surface


SCORER_FOR_SLI = {
    "share_in_word_count_band": scorers.in_word_count_band,
    "share_action_items_attendee_in_transcript": scorers.action_items_attendee_in_transcript,
    "share_without_hr_leak": scorers.no_hr_leak,
}


def load_fixtures(path: Path) -> list[dict]:
    with path.open() as fh:
        return [json.loads(line) for line in fh if line.strip()]


def main() -> int:
    plan = yaml.safe_load(Path("plan.yaml").read_text())
    fixtures = load_fixtures(Path("fixtures.jsonl"))

    report = {"surface": plan["surface"], "version": plan["version"], "rows": []}
    merge_gate_failed = False

    for row in plan["criteria"]:
        scorer = SCORER_FOR_SLI[row["sli"]]
        results = [scorer(stub_surface.run(f), f) for f in fixtures]
        sli = sum(results) / len(results)
        passed = sli >= row["slo"]
        report["rows"].append(
            {
                "id": row["id"],
                "family": row["family"],
                "sli": row["sli"],
                "measured": round(sli, 4),
                "slo": row["slo"],
                "gate": row["gate"],
                "passed": passed,
            }
        )
        if row["gate"] == "merge" and not passed:
            merge_gate_failed = True

    print(json.dumps(report, indent=2))
    return 1 if merge_gate_failed else 0


if __name__ == "__main__":
    sys.exit(main())
```

Install the one dependency and run it:

```bash
pip install pyyaml
python3 harness.py
echo "exit=$?"
```

**Expected output:** a JSON report with three rows. On the seed data, `C1` (word count) and `C2` (attendee attribution) pass at `1.0`. `C3` (HR leak) fails because `f-005` leaks the HR-tagged span. The final line should read `exit=1` — the merge-gate failed, which is the correct behaviour for the preflight check.

### Step 7 — Interpret the failing row

Re-read the `C3` row in the printed report. The fields map one-to-one onto chapter 06's criteria table:

- `family` = the five-families column.
- `sli` = the measurable quantity (chapter 02 definition).
- `measured` = the number the fixture run actually produced.
- `slo` = the pre-registered threshold from chapter 02.
- `gate` = the gate class from chapter 06 section 7.
- `passed` = whether the SLI met the SLO.

This is also the artefact a reviewer would ask for in the "offline regression suite" section of the plan (chapter 06, section 3). The harness is the "someone can rerun this suite offline from the plan alone" minimum bar made real.

### Step 8 — Simulate a regression and a fix

Flip the harness around to show both failure modes it protects you from.

**Failure mode A — regression introduced by a code change.**

Open `stub_surface.py` and change `f-003`'s `summary_words` from `205` to `150`. Save. Re-run:

```bash
python3 harness.py
echo "exit=$?"
```

Now the `C1` row drops to `0.8` (four of five fixtures in band) and fails its merge-gate SLO of `1.0`. Exit code is still `1`, but now the failing row is attributable — chapter 03's "offline owns attribution" claim is doing work: because fixtures are fixed, the only thing that changed is the code, so the regression is pinned.

Revert that line (`205`) before moving on.

**Failure mode B — fix the HR leak bug.**

In `stub_surface.py`, change `f-005`'s `summary_text_lower` to:

```python
        "summary_text_lower": "jo will draft the launch comms.",
```

Re-run:

```bash
python3 harness.py
echo "exit=$?"
```

Now all three rows pass and `exit=0`. The harness is reporting the surface meets its pre-registered merge gates.

### Step 9 — Wire the harness as a preflight CI step

Chapter 03 names "ship gates" as offline's job. The non-zero exit code from step 6 is the ship-gate signal CI wraps. This lab does not require a live CI pipeline, but the hook shape is small — commit the following snippet alongside the harness so `mod-106` has something to extend when you get there:

```yaml
# .github/workflows/preflight-eval.yml  — sketch only
name: Preflight Offline Eval
on: [pull_request]
jobs:
  preflight:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install pyyaml
      - run: python3 harness.py
```

The job fails whenever `harness.py` exits non-zero — which is exactly whenever a merge-gate SLO fails. No extra plumbing is needed for the preflight check itself; `mod-106` will layer fixture promotion, canary gates, and lineage on top.

### Step 10 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `harness.py` prints a JSON report whose `rows` match the ids in `plan.yaml` one-to-one.
- [ ] `harness.py` exits `0` on the fixed version and non-zero on the seed data (which contains the HR leak bug).
- [ ] You can name, for each row, the chapter-02 criteria family and the chapter-06 section it would appear in.
- [ ] Introducing a regression in `stub_surface.py` surfaces as the correct row failing — chapter 03's attribution property.
- [ ] The CI snippet in step 9 compiles mentally as "run `harness.py`; fail the PR if it returns non-zero."

## Where this hands off

- The harness and `plan.yaml` are the concrete form of chapter 06's section 3 ("Offline regression suite") for this surface. Keep them — `mod-106` will replace the stub surface with the real one, add fixture-hash pinning, and promote this preflight into a merge-gate CI run.
- The report shape (rows of `sli / measured / slo / gate / passed`) is the input `mod-110` (eval-data-platform slice) later ingests for history.
- The `placement: offline` field in `plan.yaml` is the chapter-03 decision rule output. The complementary rows — `online` and `deferred` — are what lab-02 and chapter 06 section 6 handle.
