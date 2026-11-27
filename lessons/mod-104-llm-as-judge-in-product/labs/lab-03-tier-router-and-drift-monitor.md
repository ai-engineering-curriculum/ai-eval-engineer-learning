# lab-03: Drive a Three-Tier Judge Router and a Drift Monitor With Deterministic Stub Judges

**Estimated duration:** 60–90 minutes
**Pairs with:** chapter 05 of this module.

## Objective

Internalise the chapter-05 routing rule and drift-monitor triage by wiring three **stub judges** (frontier / mid / OSS) with known cost-and-accuracy profiles behind a routing function, running the router against a stream of decisions, watching the per-tier spend stay inside a budget, pinning a **frozen replay set**, and then injecting each of the four chapter-05 drift sources one at a time and watching the drift monitor identify which source fired.

At the end you can read a chapter-05 "judge dashboard" off a pipeline you built yourself, and for any drift alert you can name the source (judge model version / rubric prompt / input distribution / generator) and the chapter-05 triage action without opening a textbook.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-05 wires a real **three-tier** router (frontier API + mid-tier API + self-hosted OSS judge via vLLM) against the real pipeline from exercises 01–04, writes the YAML routing config, pins a real frozen replay set from production traces, writes the drift monitor as a scheduled job, and authors the on-call runbook.
- Exercises 01–04 cover rubrics, bias controls, framework wiring, and calibration.

This lab runs entirely in stdlib Python with three ~20-line stub judges — the routing logic and the drift-triage logic are the point, not the model infrastructure.

## Prerequisites

- Chapter 05 of this module read.
- Python 3.10+ on `$PATH`.
- `pip install pyyaml` for the routing config; nothing else.
- A working directory you can write files into. **No internet, no model API keys.**

## Layout you will build

```
mod-104-lab-03/
  stub_judges.py          # frontier / mid / oss stubs with known profiles
  judge-tiers.yaml        # the routing config
  decisions.jsonl         # 60 decisions to route
  replay.jsonl            # 20 frozen replay items (separate from the above)
  router.py               # the routing + cost-budget logic
  drift_monitor.py        # the replay runner + triage
  run.py                  # CLI driver
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-104-lab-03 && cd mod-104-lab-03
```

### Step 2 — Write the three stub judges (`stub_judges.py`)

Each stub returns `{"winner": "A"|"B"|"tie", "confidence": "high"|"uncertain", "cost_usd": float, "judge_model": "<id>", "rubric_prompt_hash": "<hash>"}`. The stubs are designed so the chapter-05 routing rule has something to decide against:

- **`oss`** returns `uncertain` on items marked `hard` (simulating a weaker OSS judge that cannot resolve near-tied or policy-heavy cases); confident on everything else. Cheap.
- **`mid`** always returns `high` confidence but with a mild quality miss on ~10% of items (simulated by flipping the winner on items whose `item_id` ends in `"9"`). Moderate cost.
- **`frontier`** always returns the ground-truth winner at `high` confidence. Expensive.

```python
# stub_judges.py
"""Three stub judges with known cost/accuracy profiles for lab-03.

Not models — pure Python functions returning deterministic scores. See
exercise-05 for the real-model, real-framework version. The point of
the stubs is that the routing and the drift monitor have unambiguous
signal to react to."""

from __future__ import annotations

from typing import Literal

Winner = Literal["A", "B", "tie"]
Confidence = Literal["high", "uncertain"]

# Pin the "frozen" rubric prompt hash; the drift monitor compares.
RUBRIC_PROMPT_HASH_FROZEN = "v1-rubric-hash-abc123"


def _judge_oss(decision: dict) -> dict:
    hash_ = decision.get("rubric_prompt_hash", RUBRIC_PROMPT_HASH_FROZEN)
    truth = decision["true_winner"]
    if decision.get("difficulty") == "hard":
        return {"winner": "tie", "confidence": "uncertain",
                "cost_usd": 0.0005, "judge_model": "stub-oss-v1",
                "rubric_prompt_hash": hash_}
    return {"winner": truth, "confidence": "high",
            "cost_usd": 0.0005, "judge_model": "stub-oss-v1",
            "rubric_prompt_hash": hash_}


def _judge_mid(decision: dict) -> dict:
    hash_ = decision.get("rubric_prompt_hash", RUBRIC_PROMPT_HASH_FROZEN)
    truth = decision["true_winner"]
    # Mild quality miss: flip the winner on items whose id ends in "9".
    winner = truth
    if decision["item_id"].endswith("9") and truth in ("A", "B"):
        winner = "B" if truth == "A" else "A"
    return {"winner": winner, "confidence": "high",
            "cost_usd": 0.004, "judge_model": "stub-mid-v1",
            "rubric_prompt_hash": hash_}


def _judge_frontier(decision: dict) -> dict:
    hash_ = decision.get("rubric_prompt_hash", RUBRIC_PROMPT_HASH_FROZEN)
    return {"winner": decision["true_winner"], "confidence": "high",
            "cost_usd": 0.025, "judge_model": "stub-frontier-v1",
            "rubric_prompt_hash": hash_}


JUDGES = {
    "oss": _judge_oss,
    "mid": _judge_mid,
    "frontier": _judge_frontier,
}


def judge(tier: str, decision: dict) -> dict:
    if tier not in JUDGES:
        raise ValueError(f"unknown tier: {tier}")
    return JUDGES[tier](decision)
```

### Step 3 — Write the routing config (`judge-tiers.yaml`)

Chapter 05's "per-tier cost budget as an SLO" shape:

```yaml
# judge-tiers.yaml
tiers:
  frontier:
    model: stub-frontier-v1
    per_call_usd: 0.025
    weekly_budget_usd: 2.00
    default_for:
      - release_gate
      - calibration_rescore
  mid:
    model: stub-mid-v1
    per_call_usd: 0.004
    weekly_budget_usd: 1.00
    escalate_from_oss_on:
      - uncertain_confidence
      - tie_pairwise
  oss:
    model: stub-oss-v1
    per_call_usd: 0.0005
    weekly_budget_usd: 0.50
    default_for:
      - online_monitoring
```

### Step 4 — Seed `decisions.jsonl`

Sixty decisions with three mix labels. Each row has an `item_id`, a `true_winner`, a `decision_kind` (`online_monitoring` / `release_gate` / `calibration_rescore`), and a `difficulty` tag that controls whether the OSS stub resolves the item.

```bash
python3 - <<'PY'
import json
rows = []
for i in range(1, 51):
    truth = "A" if i % 3 != 0 else "B"
    hard  = (i % 5 == 0)
    rows.append({
        "item_id": f"d{i:03d}",
        "true_winner": truth,
        "decision_kind": "online_monitoring",
        "difficulty": "hard" if hard else "easy",
    })
# Five release-gate decisions.
for i in range(51, 56):
    rows.append({
        "item_id": f"g{i:03d}",
        "true_winner": "A" if i % 2 == 0 else "B",
        "decision_kind": "release_gate",
        "difficulty": "easy",
    })
# Five calibration-rescore decisions.
for i in range(56, 61):
    rows.append({
        "item_id": f"c{i:03d}",
        "true_winner": "A" if i % 2 == 0 else "B",
        "decision_kind": "calibration_rescore",
        "difficulty": "easy",
    })
with open("decisions.jsonl", "w") as f:
    for r in rows:
        f.write(json.dumps(r) + "\n")
print(f"wrote {len(rows)} decisions")
PY
```

**Expected:** `wrote 60 decisions`.

### Step 5 — Write the router (`router.py`)

Chapter 05's routing rule in code. The router also tracks per-tier spend and refuses to charge the frontier tier past its weekly cap (chapter 05: *"the routing rule falls back to mid-tier when the cap is reached"*).

```python
# router.py
"""Chapter-05 three-tier router + cost-budget enforcer.

Default: OSS for routine online monitoring; mid on OSS uncertainty;
frontier reserved for release_gate and calibration_rescore. Falls
back to mid when the frontier weekly cap is reached.
"""

from __future__ import annotations

import yaml
from dataclasses import dataclass, field
from pathlib import Path
from typing import Callable

from stub_judges import judge as _judge


@dataclass
class Ledger:
    spent: dict = field(default_factory=lambda: {"frontier": 0.0, "mid": 0.0, "oss": 0.0})
    cap_hits: dict = field(default_factory=lambda: {"frontier": 0, "mid": 0, "oss": 0})

    def charge(self, tier: str, usd: float) -> None:
        self.spent[tier] = round(self.spent[tier] + usd, 6)

    def remaining(self, tier: str, cap: float) -> float:
        return cap - self.spent[tier]


def load_config(path: Path) -> dict:
    return yaml.safe_load(path.read_text())


def _route_first_pass(decision: dict, cfg: dict, ledger: Ledger) -> str:
    kind = decision["decision_kind"]
    if kind in cfg["tiers"]["frontier"]["default_for"]:
        cap = cfg["tiers"]["frontier"]["weekly_budget_usd"]
        if ledger.remaining("frontier", cap) >= cfg["tiers"]["frontier"]["per_call_usd"]:
            return "frontier"
        ledger.cap_hits["frontier"] += 1
        return "mid"  # chapter-05 fallback when frontier cap is reached
    return "oss"


def _route_escalation(oss_result: dict) -> str:
    if oss_result["confidence"] == "uncertain" or oss_result["winner"] == "tie":
        return "mid"
    return "oss"


def judge_one(decision: dict, cfg: dict, ledger: Ledger,
              judge_fn: Callable[[str, dict], dict] = _judge) -> dict:
    """Run a single decision through the router; return the final judgement
    plus the tier that produced it. Logs cost against the ledger."""
    tier = _route_first_pass(decision, cfg, ledger)
    result = judge_fn(tier, decision)
    ledger.charge(tier, result["cost_usd"])

    # Chapter-05 escalation step: if we started on OSS and it's uncertain,
    # kick up to mid.
    if tier == "oss":
        new_tier = _route_escalation(result)
        if new_tier != tier:
            result = judge_fn(new_tier, decision)
            ledger.charge(new_tier, result["cost_usd"])
            tier = new_tier

    return {
        **result,
        "app.judge_tier": tier,
        "item_id": decision["item_id"],
        "decision_kind": decision["decision_kind"],
        "true_winner": decision["true_winner"],
    }


def judge_batch(decisions: list[dict], cfg: dict,
                judge_fn: Callable[[str, dict], dict] = _judge) -> tuple[list[dict], Ledger]:
    ledger = Ledger()
    results = [judge_one(d, cfg, ledger, judge_fn) for d in decisions]
    return results, ledger
```

### Step 6 — Seed `replay.jsonl` (the frozen replay set)

Twenty items. Separate from `decisions.jsonl` — chapter 05 is explicit: *"Not the calibration gold set — a separate replay set so calibration can drift too and you notice."*

```bash
python3 - <<'PY'
import json
rows = []
for i in range(1, 21):
    rows.append({
        "item_id": f"replay{i:03d}",
        "true_winner": "A" if i % 2 == 0 else "B",
        "decision_kind": "online_monitoring",
        "difficulty": "easy",
    })
with open("replay.jsonl", "w") as f:
    for r in rows:
        f.write(json.dumps(r) + "\n")
print("wrote 20 replay items")
PY
```

### Step 7 — Write the drift monitor (`drift_monitor.py`)

The monitor runs the current production judge over the replay set, compares the result against a pinned baseline, and triages per chapter 05 Table of Triggers.

```python
# drift_monitor.py
"""Chapter-05 drift monitor and triage.

Runs the current production judge over the frozen replay set, compares
each result against a pinned baseline (judge_model, rubric_prompt_hash,
per-item winner), and names which of the four chapter-05 drift sources
fired.
"""

from __future__ import annotations

import json
from pathlib import Path

from stub_judges import judge as _judge


def pin_baseline(replay: list[dict], cfg: dict, tier: str = "oss") -> list[dict]:
    """Run the current-production judge over the replay set once and
    store the result as the frozen baseline."""
    pinned = []
    for item in replay:
        r = _judge(tier, item)
        pinned.append({
            "item_id": item["item_id"],
            "winner": r["winner"],
            "judge_model": r["judge_model"],
            "rubric_prompt_hash": r["rubric_prompt_hash"],
        })
    return pinned


def run_monitor(replay: list[dict], baseline: list[dict], tier: str = "oss") -> dict:
    """Re-run the current-production judge over the replay set and
    report the chapter-05 drift triage."""
    current = []
    for item in replay:
        r = _judge(tier, item)
        current.append({
            "item_id": item["item_id"],
            "winner": r["winner"],
            "judge_model": r["judge_model"],
            "rubric_prompt_hash": r["rubric_prompt_hash"],
        })

    # Pair up for comparison.
    disagreements = sum(
        1 for b, c in zip(baseline, current) if b["winner"] != c["winner"]
    )
    disagreement_rate = disagreements / max(1, len(baseline))

    judge_model_changed = baseline[0]["judge_model"] != current[0]["judge_model"]
    rubric_hash_changed = (
        baseline[0]["rubric_prompt_hash"] != current[0]["rubric_prompt_hash"]
    )

    # Chapter-05 drift-source triage.
    triage = []
    if rubric_hash_changed:
        triage.append({
            "source": "2-rubric-prompt-template",
            "evidence": f"rubric_prompt_hash {baseline[0]['rubric_prompt_hash']} -> "
                        f"{current[0]['rubric_prompt_hash']}",
            "action": "re-calibrate or roll back the rubric; re-pin the replay baseline",
        })
    if judge_model_changed:
        triage.append({
            "source": "1-judge-model-version",
            "evidence": f"judge_model {baseline[0]['judge_model']} -> "
                        f"{current[0]['judge_model']}",
            "action": "frontier-upgrade playbook: pin, side-by-side, re-calibrate, swap",
        })
    if disagreement_rate > 0 and not (judge_model_changed or rubric_hash_changed):
        # Chapter-05: distinguishes true judge drift from population drift.
        # Here, if neither model nor rubric changed but outputs did, suspect
        # drift source (3) input distribution or (4) generator model.
        triage.append({
            "source": "3-or-4-population-or-generator",
            "evidence": f"{disagreements}/{len(baseline)} replay items disagree "
                        f"with no judge-model or rubric-hash change",
            "action": "compare production input distribution and generator "
                     "model version to pinned baseline; refresh calibration set "
                     "if distribution shifted; do NOT re-calibrate if generator alone changed",
        })

    return {
        "n": len(baseline),
        "disagreements": disagreements,
        "disagreement_rate": round(disagreement_rate, 4),
        "judge_model_changed": judge_model_changed,
        "rubric_hash_changed": rubric_hash_changed,
        "triage": triage,
        "alert": disagreement_rate > 0.0 or judge_model_changed or rubric_hash_changed,
    }


def load_jsonl(path: Path) -> list[dict]:
    return [json.loads(line) for line in path.read_text().splitlines() if line.strip()]
```

### Step 8 — Write the CLI driver (`run.py`)

```python
# run.py
"""Run the three-tier router over decisions.jsonl, print the dashboard,
then pin the drift-monitor baseline and run it once to confirm no
drift. Drift-source injection happens in steps 10-13.
"""

import json
from collections import Counter
from pathlib import Path

from drift_monitor import load_jsonl, pin_baseline, run_monitor
from router import judge_batch, load_config


def dashboard(results: list[dict], ledger) -> dict:
    tier_counts = Counter(r["app.judge_tier"] for r in results)
    hits = sum(1 for r in results if r["winner"] == r["true_winner"])
    return {
        "n_decisions": len(results),
        "tier_counts": dict(tier_counts),
        "spend_usd": ledger.spent,
        "cap_hits": ledger.cap_hits,
        "ground_truth_agreement": round(hits / len(results), 4),
    }


def main() -> int:
    cfg = load_config(Path("judge-tiers.yaml"))
    decisions = load_jsonl(Path("decisions.jsonl"))
    replay = load_jsonl(Path("replay.jsonl"))

    print("## Routing + cost dashboard\n")
    results, ledger = judge_batch(decisions, cfg)
    board = dashboard(results, ledger)
    print(json.dumps(board, indent=2))
    print()

    print("## Drift monitor (baseline + first run)\n")
    baseline = pin_baseline(replay, cfg, tier="oss")
    Path("replay-baseline.json").write_text(json.dumps(baseline, indent=2))
    monitor = run_monitor(replay, baseline, tier="oss")
    print(json.dumps(monitor, indent=2))
    return 0


if __name__ == "__main__":
    import sys
    sys.exit(main())
```

### Step 9 — Run the baseline

```bash
pip install pyyaml
python3 run.py
```

**Expected (deterministic):**

- **`tier_counts`.** Most online-monitoring decisions land on `oss`; the ten "hard" items (ids ending in multiples of 5) escalate to `mid`; all five `release_gate` and all five `calibration_rescore` decisions land on `frontier`.
- **`spend_usd`.** `frontier` spend is ~10 × $0.025 = $0.25; `mid` spend covers the ten OSS-uncertain escalations (~$0.04); `oss` spend covers the ~50 OSS first-pass calls (~$0.025). All well under the per-tier caps.
- **`cap_hits`.** Zero — none of the caps were reached on this traffic mix.
- **`ground_truth_agreement`.** High. Most errors are the `mid` stub's deliberate flip on `d009` / `d019` / `d029` / `d039` / `d049`. The frontier tier never misses.
- **Drift monitor baseline.** `disagreements == 0`, `judge_model_changed == False`, `rubric_hash_changed == False`, `alert == False`. The monitor is quiet because nothing has drifted.

### Step 10 — Inject drift source 2 (rubric prompt template change)

Chapter 05 drift source 2: a rubric edit. Re-run the monitor with a different `rubric_prompt_hash` baked into the replay items.

```bash
python3 - <<'PY'
import json
from pathlib import Path
from drift_monitor import load_jsonl, run_monitor
replay = load_jsonl(Path("replay.jsonl"))
baseline = json.loads(Path("replay-baseline.json").read_text())
# Inject a rubric edit: the live production rubric hash is now different.
for item in replay:
    item["rubric_prompt_hash"] = "v2-rubric-hash-xyz789"
print(json.dumps(run_monitor(replay, baseline, tier="oss"), indent=2))
PY
```

**Expected:** `rubric_hash_changed: true`, `triage` contains a `2-rubric-prompt-template` entry with a "re-calibrate or roll back" action. This is the chapter-05 "same judge, different scores" case; the mitigation is a re-calibration, not an incident.

### Step 11 — Inject drift source 1 (judge model version change)

Simulate the frontier vendor silently rolling the snapshot behind the alias.

```bash
python3 - <<'PY'
import json
from pathlib import Path
from drift_monitor import load_jsonl, pin_baseline, run_monitor
from router import load_config
cfg = load_config(Path("judge-tiers.yaml"))
replay = load_jsonl(Path("replay.jsonl"))
baseline = pin_baseline(replay, cfg, tier="oss")

# Swap the oss stub to a "v2" with a materially different label.
import stub_judges
original = stub_judges._judge_oss
def v2(decision):
    r = original(decision)
    r["judge_model"] = "stub-oss-v2"
    return r
stub_judges._judge_oss = v2
stub_judges.JUDGES["oss"] = v2

print(json.dumps(run_monitor(replay, baseline, tier="oss"), indent=2))

# Restore for the next step.
stub_judges._judge_oss = original
stub_judges.JUDGES["oss"] = original
PY
```

**Expected:** `judge_model_changed: true`, `triage` contains a `1-judge-model-version` entry with the chapter-05 **frontier-upgrade playbook** named as the action: *pin, side-by-side, re-calibrate, swap*. Chapter 05 is explicit that this is where re-calibration is mandatory — the pre-upgrade number does not defend the post-upgrade pipeline.

### Step 12 — Inject drift source 3/4 (population or generator drift)

Simulate a change that affects *outputs* but leaves the judge and the rubric hash unchanged — this is chapter 05 drift source (3) input distribution or (4) generator model. The monitor must **not** blame the judge.

```bash
python3 - <<'PY'
import json
from pathlib import Path
from drift_monitor import load_jsonl, pin_baseline, run_monitor
from router import load_config
cfg = load_config(Path("judge-tiers.yaml"))
replay = load_jsonl(Path("replay.jsonl"))
baseline = pin_baseline(replay, cfg, tier="oss")

# Simulate upstream (generator/input) drift: ground-truth winners flip
# on ~30% of replay items, but the judge model and rubric hash are UNCHANGED.
for i, item in enumerate(replay):
    if i % 3 == 0:
        item["true_winner"] = "A" if item["true_winner"] == "B" else "B"

print(json.dumps(run_monitor(replay, baseline, tier="oss"), indent=2))
PY
```

**Expected:** `judge_model_changed: false`, `rubric_hash_changed: false`, `disagreement_rate > 0`, `triage` contains a `3-or-4-population-or-generator` entry with the chapter-05 action — compare input distribution and generator version, refresh the calibration set only if the *distribution* shifted, **do NOT re-calibrate** if only the generator changed. The monitor keeps you from re-calibrating the judge against a generator regression and masking it.

### Step 13 — Breach the frontier budget cap

Chapter 05 failure mode: frontier tier consumes the whole budget. Shrink the frontier cap to $0.05 (two calls' worth) and re-run.

```bash
python3 - <<'PY'
import yaml
from pathlib import Path
cfg = yaml.safe_load(Path("judge-tiers.yaml").read_text())
cfg["tiers"]["frontier"]["weekly_budget_usd"] = 0.05   # was 2.00
Path("judge-tiers-tight.yaml").write_text(yaml.safe_dump(cfg, sort_keys=False))
PY

python3 - <<'PY'
import json
from pathlib import Path
from drift_monitor import load_jsonl
from router import judge_batch, load_config
cfg = load_config(Path("judge-tiers-tight.yaml"))
decisions = load_jsonl(Path("decisions.jsonl"))
results, ledger = judge_batch(decisions, cfg)
from collections import Counter
print(json.dumps({
    "tier_counts": dict(Counter(r["app.judge_tier"] for r in results)),
    "spend_usd": ledger.spent,
    "cap_hits": ledger.cap_hits,
}, indent=2))
PY
```

**Expected:** `frontier` serves only the first two release-gate/calibration-rescore decisions; the rest fall back to `mid` (chapter-05's rule). `cap_hits.frontier` is **8** (the remaining 8 frontier-default decisions were downgraded). Chapter 05 is explicit: *"the routing rule falls back to mid-tier when the cap is reached"* — the pipeline degrades gracefully instead of 500-ing.

### Step 14 — Write the on-call runbook

Chapter 05 is clear that the drift monitor's output has to tell an on-call engineer what to do. Author `runbook.md` in the lab directory with one section per drift source — each section lists the signal, the attribution rule, and the next action:

```markdown
# Judge drift on-call runbook

## Signal: `triage[].source == "1-judge-model-version"`
- Evidence: `judge_model` field in replay baseline differs from current run.
- Action: Frontier-upgrade playbook (chapter 05) — pin, side-by-side on
  replay AND calibration gold set, re-calibrate against human labels,
  publish new statistic, swap, keep incumbent for one drift-monitor cycle.

## Signal: `triage[].source == "2-rubric-prompt-template"`
- Evidence: `rubric_prompt_hash` differs; the chapter-01 prompt template
  was edited.
- Action: Treat as a code-review miss. Either (a) roll back the rubric,
  or (b) re-run chapter-04 calibration and update the SLI threshold.

## Signal: `triage[].source == "3-or-4-population-or-generator"`
- Evidence: Replay disagreements rose with no model or rubric change.
- Action: Compare production input-distribution stats and generator-model
  version to the pinned baseline. If input distribution shifted: refresh
  the calibration set (chapter 04 triggers, row 4). If only the generator
  changed: do NOT re-calibrate the judge — the judge is faithfully
  reporting a generator regression. Route the alert to the generator team.

## Signal: `cap_hits.frontier > 0`
- Evidence: Frontier weekly cap reached; downgraded decisions landed on mid.
- Action: Audit the routing config; a per-criterion default is probably
  over-reaching into frontier when OSS + mid would suffice. Chapter 05's
  "frontier-overspend blocking online eval" failure mode.
```

### Step 15 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `python3 run.py` prints a `tier_counts` dict where most online-monitoring items land on `oss`, the "hard" items land on `mid`, and release-gate + calibration-rescore items land on `frontier`.
- [ ] The baseline drift-monitor run returns `alert: false` and empty `triage`.
- [ ] Step 10's rubric-hash injection triggers a `2-rubric-prompt-template` triage entry naming "re-calibrate or roll back".
- [ ] Step 11's judge-model swap triggers a `1-judge-model-version` entry naming the frontier-upgrade playbook.
- [ ] Step 12's upstream-flip injection triggers a `3-or-4-population-or-generator` entry whose action is **not** "re-calibrate the judge".
- [ ] Step 13's tight-cap run reports non-zero `cap_hits.frontier` and the downgraded decisions fall back to `mid`, not to an exception.
- [ ] You can state why the chapter-05 monitor separates (1)+(2) from (3)+(4): calling drift source (4) a "judge drift" and re-calibrating would mask a generator regression.
- [ ] Your `runbook.md` has one section per drift source and each section lists signal → attribution → action.

## Where this hands off

- `judge-tiers.yaml` is the exact shape exercise-05 asks for on your real pipeline, with real per-call costs and real snapshot ids in place of the stubs.
- `router.py` + `drift_monitor.py` are the shape mod-107's online-eval loop reads at runtime — the `app.judge_tier` attribute lands on every EVALUATOR span per mod-102's schema, and the drift triage lines are what mod-107's alerting pages on.
- `runbook.md` is the on-call artefact mod-112's governance posture asks for; exercise-05 extends this with the chapter-05 "judge dashboard" screenshot block.
- The lab's clear separation of judge drift (1+2) from population / generator drift (3+4) is exactly the escalation handoff that keeps the judge-vs-generator ownership line clean when mod-107's alerting fires on an incident.
