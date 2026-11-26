# lab-01: Watch a Per-Step Scorer Fire on Surgical Edits to a Canned Trajectory

**Estimated duration:** 60–90 minutes
**Pairs with:** chapters 02 and 03 of this module.

## Objective

Internalise the chapter-02 verdict vocabulary — `wrong_tool`, `wrong_args`, `ordering_violation`, `redundant_call`, `unauthorised_tool` — by doing *one surgical edit at a time* to a known-good trajectory and watching a small scorer flag exactly the verdict the edit was designed to produce. The scorer is provided; you are not building it from scratch (exercise-01 is where you do that). The point of this lab is to make the mapping between "what the model did" and "which verdict the scorer emits" mechanical in your head.

At the end you can look at any flagged verdict in a production dashboard and name the shape of the step-level evidence that produced it — the vocabulary chapter 02 fixes and chapter 03 rolls up into the partial-credit rubric.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-01 asks you to build the per-step scorer end-to-end with tests, a CLI, and docs.
- Exercise-02 assembles the chapter-03 partial-credit rubric on top of exercise-01.
- Exercise-03 wraps all of it inside Inspect against a real model.
- Exercises 04–05 audit public benchmarks and build the replay context.

This lab gives you a working scorer and a canned known-good trajectory, and drills the "one edit, one verdict" discipline the rubric vocabulary depends on.

## Prerequisites

- Chapters 02 and 03 read.
- Python 3.10+ on `$PATH` (`python3 --version`) — 3.9+ works, but the `graphlib` stdlib module was added in 3.9 and we rely on it.
- A working directory you can write files into. No internet, no model API keys, no Docker — the scorer is pure stdlib Python and the trajectories are JSON.

## Layout you will build

```
mod-103-lab-01/
  fixtures/
    reference.json              # DAG + argument rules + allowed_set for the task family
    trajectory_good.json        # a passing trajectory (step 0)
    trajectory_wrong_tool.json  # a surgical edit (step 4)
    trajectory_bad_args.json    # a surgical edit (step 5)
    trajectory_bad_order.json   # a surgical edit (step 6)
    trajectory_redundant.json   # a surgical edit (step 7)
  scorer.py                     # the per-step scorer
  run.py                        # the CLI driver that prints the verdict
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-103-lab-01/fixtures && cd mod-103-lab-01
```

Confirm you are in an empty directory with `fixtures/` inside:

```bash
ls -a
```

**Expected:** `.`, `..`, `fixtures`.

### Step 2 — Author the reference trajectory

The reference encodes the DAG, the per-argument rules, and the allowed / required sets for the running `order_status_not_shipped_policy` task family from chapter 01. Create `fixtures/reference.json`:

```json
{
  "task_family": "order_status_not_shipped_policy",
  "reference_version": "v3.2-lab01",
  "form": "DAG",
  "required_set": ["docs.search", "open_ticket"],
  "allowed_set":  ["docs.search", "crm.lookup", "open_ticket"],
  "edges": [
    ["docs.search", "open_ticket"],
    ["crm.lookup",  "open_ticket"]
  ],
  "argument_rules": {
    "docs.search": {
      "query": {"level": 3, "rule": "token_set_includes", "tokens": ["order", "status"]},
      "top_k": {"level": 1, "rule": "type_int"}
    },
    "open_ticket": {
      "order_id": {"level": 1, "rule": "type_int"},
      "reason":   {"level": 2, "rule": "in_set",
                   "values": ["no_shipping_update", "shipping_delay", "delivery_exception"]}
    },
    "crm.lookup": {
      "order_id": {"level": 1, "rule": "type_int"}
    }
  }
}
```

### Step 3 — Author the known-good trajectory

Create `fixtures/trajectory_good.json`. This is a passing trajectory — it should flag nothing when the scorer runs.

```json
{
  "trajectory_id": "traj-good",
  "task_family":   "order_status_not_shipped_policy",
  "steps": [
    {"step_id": 1, "span_id": "s1", "kind": "LLM",  "app.step": "plan"},
    {"step_id": 2, "span_id": "s2", "kind": "TOOL",
     "tool_name": "docs.search",
     "arguments": {"query": "order 12345 status", "top_k": 3}},
    {"step_id": 3, "span_id": "s3", "kind": "LLM",  "app.step": "plan.2"},
    {"step_id": 4, "span_id": "s4", "kind": "TOOL",
     "tool_name": "crm.lookup",
     "arguments": {"order_id": 12345}},
    {"step_id": 5, "span_id": "s5", "kind": "LLM",  "app.step": "plan.3"},
    {"step_id": 6, "span_id": "s6", "kind": "TOOL",
     "tool_name": "open_ticket",
     "arguments": {"order_id": 12345, "reason": "no_shipping_update"}},
    {"step_id": 7, "span_id": "s7", "kind": "LLM",  "app.step": "reply"}
  ]
}
```

### Step 4 — Write the per-step scorer (`scorer.py`)

The scorer implements the chapter-02 name, argument, and ordering matchers. Keep it under 200 lines; stdlib only.

```python
# scorer.py
"""Per-step scorer for the lab-01 trajectory format.

Chapter 02's three-axis per-step verdict in code form: name matcher with
retry collapsing + NFKC/casefold normalisation, argument matchers at
Levels 1 (structural), 2 (exact-value / set-membership), and 3
(semantic-equivalence by token-set), and a partial-order (topological)
ordering matcher built on graphlib.TopologicalSorter. No judges; no
Inspect; no model calls."""

import json
import re
import unicodedata
from graphlib import TopologicalSorter, CycleError
from pathlib import Path


# --- normalisation ---------------------------------------------------------

def _norm_name(name: str) -> str:
    return unicodedata.normalize("NFKC", name).casefold()


def _tokens(s: str) -> set:
    return {t for t in re.findall(r"[a-z0-9]+", s.lower()) if t not in {"of", "the", "a"}}


# --- retry collapsing ------------------------------------------------------

def _collapse_retries(tool_steps: list) -> list:
    """Chapter 02: consecutive identical (tool_name, arguments) calls that
    differ only in success status collapse to one. Retention of the retry
    count is the budget scorer's job (chapter 04)."""
    out = []
    for s in tool_steps:
        key = (_norm_name(s["tool_name"]), json.dumps(s["arguments"], sort_keys=True))
        if out and out[-1]["_key"] == key:
            out[-1].setdefault("retry_count", 0)
            out[-1]["retry_count"] += 1
            continue
        out.append({**s, "_key": key})
    return out


# --- argument matcher ------------------------------------------------------

def _match_args(observed: dict, rules: dict) -> dict:
    """Return {level_used, match, details}. `level_used` is the highest
    level the chapter's ladder needed to confirm the match."""
    if observed.keys() != rules.keys():
        missing = sorted(set(rules) - set(observed))
        extra   = sorted(set(observed) - set(rules))
        return {"level_used": 1, "match": False,
                "details": {"missing_keys": missing, "extra_keys": extra}}

    highest_level = 1
    for key, rule in rules.items():
        val = observed[key]
        if rule["rule"] == "type_int" and not isinstance(val, int):
            return {"level_used": 1, "match": False,
                    "details": {"key": key, "expected_type": "int", "got": type(val).__name__}}
        if rule["rule"] == "in_set":
            highest_level = max(highest_level, 2)
            if val not in rule["values"]:
                return {"level_used": 2, "match": False,
                        "details": {"key": key, "expected_in": rule["values"], "got": val}}
        if rule["rule"] == "token_set_includes":
            highest_level = max(highest_level, 3)
            need = {t.lower() for t in rule["tokens"]}
            have = _tokens(str(val))
            if not need.issubset(have):
                return {"level_used": 3, "match": False,
                        "details": {"key": key, "missing_tokens": sorted(need - have)}}
    return {"level_used": highest_level, "match": True, "details": {}}


# --- ordering matcher ------------------------------------------------------

def _match_order(observed_names: list, reference: dict) -> dict:
    """Chapter 02's partial-order matcher via graphlib.TopologicalSorter.
    Returns {matcher, match, violated_edge}."""
    sorter = TopologicalSorter()
    for u, v in reference["edges"]:
        sorter.add(v, u)  # edge u -> v means u is a prerequisite of v
    try:
        sorter.prepare()
    except CycleError as exc:
        return {"matcher": "topological", "match": False, "violated_edge": f"cycle:{exc}"}

    seen = set()
    for name in observed_names:
        name_n = _norm_name(name)
        seen.add(name_n)
        for u, v in reference["edges"]:
            if _norm_name(v) == name_n and _norm_name(u) not in seen:
                return {"matcher": "topological", "match": False,
                        "violated_edge": f"{u} -> {v}"}
    return {"matcher": "topological", "match": True, "violated_edge": None}


# --- redundant-call detector ----------------------------------------------

def _redundant(tool_steps: list, reference: dict) -> set:
    """A second call to the same tool with a semantically-equivalent
    argument is `redundant_call` (chapter 02's running example — the
    second docs.search). Returns the set of step_ids to flag."""
    flagged = set()
    seen_keys = {}
    for s in tool_steps:
        rules = reference["argument_rules"].get(_norm_name(s["tool_name"]))
        if not rules:
            continue
        # Build a comparator key from the argument-match semantics so
        # queries that differ only in token order look equal.
        parts = []
        for key, rule in rules.items():
            val = s["arguments"].get(key)
            if rule["rule"] == "token_set_includes":
                parts.append((key, frozenset(_tokens(str(val)))))
            else:
                parts.append((key, val))
        k = (_norm_name(s["tool_name"]), tuple(parts))
        if k in seen_keys:
            flagged.add(s["step_id"])
        else:
            seen_keys[k] = s["step_id"]
    return flagged


# --- top-level per-step scorer --------------------------------------------

def score(trajectory: dict, reference: dict) -> dict:
    tool_steps = [s for s in trajectory["steps"] if s["kind"] == "TOOL"]
    tool_steps = _collapse_retries(tool_steps)

    redundant_step_ids = _redundant(tool_steps, reference)

    observed_names = [s["tool_name"] for s in tool_steps]
    order_v = _match_order(observed_names, reference)

    allowed = {_norm_name(n) for n in reference["allowed_set"]}
    required = {_norm_name(n) for n in reference["required_set"]}

    per_step = []
    flagged = []
    for s in tool_steps:
        name = _norm_name(s["tool_name"])
        name_ok = name in allowed  # allowed-but-not-required is still ok
        if not name_ok:
            flagged.append("unauthorised_tool")

        rules = reference["argument_rules"].get(s["tool_name"], {})
        args_v = _match_args(s["arguments"], rules) if rules else \
                 {"level_used": None, "match": name_ok, "details": {}}

        step_verdict = "ok"
        if not name_ok:
            step_verdict = "unauthorised_tool"
        elif not args_v["match"]:
            step_verdict = "wrong_args"
            flagged.append("wrong_args")
        elif s["step_id"] in redundant_step_ids:
            step_verdict = "redundant_call"
            flagged.append("redundant_call")

        per_step.append({
            "step_id": s["step_id"],
            "span_id": s["span_id"],
            "tool_name": {"actual": s["tool_name"], "match": name_ok},
            "arguments": args_v,
            "order": {"matcher": "topological", "match": order_v["match"]},
            "retry_count": s.get("retry_count", 0),
            "verdict": step_verdict,
        })

    if not order_v["match"]:
        flagged.append("ordering_violation")

    # missing_required_tool is a trajectory-level verdict.
    observed_tool_set = {_norm_name(n) for n in observed_names}
    for req in required:
        if req not in observed_tool_set:
            flagged.append("missing_required_tool")

    name_pass_rate = sum(1 for v in per_step if v["tool_name"]["match"]) / max(1, len(per_step))
    args_pass_rate = sum(1 for v in per_step if v["arguments"]["match"]) / max(1, len(per_step))
    order_pass_rate = 1.0 if order_v["match"] else 0.0

    return {
        "trajectory_id": trajectory["trajectory_id"],
        "task_family":   trajectory["task_family"],
        "reference_version": reference["reference_version"],
        "per_step": per_step,
        "summary": {
            "name_pass_rate":  round(name_pass_rate, 3),
            "args_pass_rate":  round(args_pass_rate, 3),
            "order_pass_rate": order_pass_rate,
            "flagged_verdicts": sorted(set(flagged)),
            "violated_edge":   order_v["violated_edge"],
        },
    }


if __name__ == "__main__":  # pragma: no cover
    import sys
    traj = json.loads(Path(sys.argv[1]).read_text())
    ref  = json.loads(Path(sys.argv[2]).read_text())
    print(json.dumps(score(traj, ref), indent=2))
```

### Step 5 — Write the one-screen driver (`run.py`)

```python
# run.py
"""Prints `summary.flagged_verdicts` for a trajectory, one line.

Use this as a smoke test before dumping the full JSON."""

import json
import sys
from pathlib import Path

from scorer import score


def main() -> int:
    traj = json.loads(Path(sys.argv[1]).read_text())
    ref  = json.loads(Path(sys.argv[2]).read_text())
    v = score(traj, ref)
    print(f"{Path(sys.argv[1]).name}: "
          f"name={v['summary']['name_pass_rate']} "
          f"args={v['summary']['args_pass_rate']} "
          f"order={v['summary']['order_pass_rate']} "
          f"flagged={v['summary']['flagged_verdicts']}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### Step 6 — Run the scorer on the good trajectory (baseline)

```bash
python3 run.py fixtures/trajectory_good.json fixtures/reference.json
```

**Expected output:**

```
trajectory_good.json: name=1.0 args=1.0 order=1.0 flagged=[]
```

If `flagged` is not empty on the good trajectory, re-check the scorer or the fixture — the lab's whole point is one-edit / one-verdict, which requires a clean baseline.

### Step 7 — Surgical edit #1: wrong tool

Copy the good trajectory and change one thing — the tool name at step 2 — from `docs.search` to `sql_query` (chapter 01's golden shortcut):

```bash
python3 - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("fixtures/trajectory_good.json").read_text())
t["trajectory_id"] = "traj-wrong-tool"
t["steps"][1]["tool_name"] = "sql_query"
Path("fixtures/trajectory_wrong_tool.json").write_text(json.dumps(t, indent=2))
PY
python3 run.py fixtures/trajectory_wrong_tool.json fixtures/reference.json
```

**Expected output** (the exact `args` figure depends on how the scorer treats the unknown tool; the key signal is the three-way `flagged` list):

```
trajectory_wrong_tool.json: name=0.667 args=0.667 order=0.0 flagged=['missing_required_tool', 'ordering_violation', 'unauthorised_tool']
```

Why: `sql_query` is not in `allowed_set`, so the step verdict is `unauthorised_tool`; the required `docs.search` is now missing, so `missing_required_tool` fires at trajectory level; and the DAG edge `docs.search → open_ticket` is violated because `docs.search` never ran before `open_ticket` — so the topological matcher rejects the sequence. One surgical edit, three distinct verdicts, each attributable.

### Step 8 — Surgical edit #2: wrong argument

Keep the tool right; break the argument. Change `open_ticket.reason` from an allowed value to one not in the enumerated set:

```bash
python3 - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("fixtures/trajectory_good.json").read_text())
t["trajectory_id"] = "traj-bad-args"
t["steps"][5]["arguments"]["reason"] = "customer_vibes"
Path("fixtures/trajectory_bad_args.json").write_text(json.dumps(t, indent=2))
PY
python3 run.py fixtures/trajectory_bad_args.json fixtures/reference.json
```

**Expected output:**

```
trajectory_bad_args.json: name=1.0 args=0.667 order=1.0 flagged=['wrong_args']
```

The Level-2 `in_set` matcher rejects `"customer_vibes"`; `wrong_args` fires on exactly that step; the name axis and the order axis are untouched. Attribution routes the fix to the planner prompt (why did it pick a free-form reason?), not to the tool list.

Dump the full verdict to see which step and which level flagged:

```bash
python3 scorer.py fixtures/trajectory_bad_args.json fixtures/reference.json
```

Look for the `per_step[5]` entry: `arguments.level_used == 2`, `arguments.match == false`, `arguments.details.key == "reason"`, `verdict == "wrong_args"`.

### Step 9 — Surgical edit #3: wrong order

Move the `open_ticket` step earlier in the sequence — before `docs.search` — to violate the DAG edge:

```bash
python3 - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("fixtures/trajectory_good.json").read_text())
t["trajectory_id"] = "traj-bad-order"
# Swap docs.search (step_id 2) and open_ticket (step_id 6).
steps = t["steps"]
tool_indices = [i for i, s in enumerate(steps) if s["kind"] == "TOOL"]
docs_idx = next(i for i in tool_indices if steps[i]["tool_name"] == "docs.search")
open_idx = next(i for i in tool_indices if steps[i]["tool_name"] == "open_ticket")
steps[docs_idx], steps[open_idx] = steps[open_idx], steps[docs_idx]
Path("fixtures/trajectory_bad_order.json").write_text(json.dumps(t, indent=2))
PY
python3 run.py fixtures/trajectory_bad_order.json fixtures/reference.json
```

**Expected output:**

```
trajectory_bad_order.json: name=1.0 args=1.0 order=0.0 flagged=['ordering_violation']
```

Both tools were called with correct names and correct arguments; only the order is wrong. The topological matcher names the violated edge; run the full scorer to see it:

```bash
python3 scorer.py fixtures/trajectory_bad_order.json fixtures/reference.json | grep violated_edge
```

**Expected:** a line naming the edge, e.g. `"violated_edge": "docs.search -> open_ticket"`.

### Step 10 — Surgical edit #4: redundant call

Duplicate the `docs.search` step with a semantically-equivalent query (chapter 01's second `docs.search` with a differently-worded query that carries the same token set). The redundant-call detector in Step 4 uses the Level-3 token-set comparator so this should flag:

```bash
python3 - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("fixtures/trajectory_good.json").read_text())
t["trajectory_id"] = "traj-redundant"
# Insert a second docs.search with a different-but-equivalent query.
redundant = {"step_id": 99, "span_id": "s99", "kind": "TOOL",
             "tool_name": "docs.search",
             "arguments": {"query": "status of order 12345", "top_k": 3}}
# Insert after the first docs.search (which is step_id 2, index 1).
t["steps"].insert(2, redundant)
Path("fixtures/trajectory_redundant.json").write_text(json.dumps(t, indent=2))
PY
python3 run.py fixtures/trajectory_redundant.json fixtures/reference.json
```

**Expected output:**

```
trajectory_redundant.json: name=1.0 args=1.0 order=1.0 flagged=['redundant_call']
```

The second `docs.search` is a valid call in isolation (`name_ok`, `args_ok`, order respects the DAG) but it carries no new information — the Level-3 token-set rule treats `"order 12345 status"` and `"status of order 12345"` as semantically equivalent (chapter 02's exact example). `redundant_call` is the only new verdict.

### Step 11 — Checkpoint: the surgical-edit discipline

For each of the four edits you made, answer these without re-running the scorer:

- Which **one** thing changed in the trajectory JSON?
- Which **primary** verdict fired?
- Which **secondary** verdicts, if any, followed (e.g., `missing_required_tool` as a knock-on of `unauthorised_tool`)?
- Which axis (`name`, `args`, `order`) is the attribution block pointing at?
- If you were on-call and the dashboard showed this one verdict, which of chapter 02's four fix locations would you route to — tool catalog, planner prompt, DAG reference, or the harness retry policy?

If you cannot answer the four questions for a given verdict, re-read the chapter-02 section on that axis and re-run the scorer on the matching fixture.

### Step 12 — Stretch: compose two edits, confirm both verdicts fire

Combine `redundant_call` on a docs.search step with `wrong_args` on the open_ticket step. Start from the redundant-call fixture and additionally break `open_ticket.reason`:

```bash
python3 - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("fixtures/trajectory_redundant.json").read_text())
t["trajectory_id"] = "traj-compose"
for s in t["steps"]:
    if s["kind"] == "TOOL" and s["tool_name"] == "open_ticket":
        s["arguments"]["reason"] = "customer_vibes"
Path("fixtures/trajectory_compose.json").write_text(json.dumps(t, indent=2))
PY
python3 run.py fixtures/trajectory_compose.json fixtures/reference.json
```

**Expected:** `flagged` contains both `redundant_call` **and** `wrong_args`. The two verdicts attach to two different steps and two different axes; the scorer does not collapse them into one. Chapter 02 is explicit that the matchers must not short-circuit each other — if either verdict is missing, re-read the "Sub-scores rolled into a per-step verdict" section and re-check `scorer.py`.

### Step 13 — Final checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `python3 run.py fixtures/trajectory_good.json fixtures/reference.json` emits `flagged=[]`.
- [ ] Each of the four surgical-edit fixtures produces the expected primary verdict in isolation.
- [ ] The full scorer output (`python3 scorer.py ...`) records `arguments.level_used` on every tool step — Level 1 for integer types, Level 2 for `in_set`, Level 3 for token-set.
- [ ] The ordering matcher names the violated edge, not just a boolean, on the bad-order fixture.
- [ ] The composed-edit fixture flags **both** verdicts; neither matcher short-circuits the other.
- [ ] You can state, for each flagged verdict, the chapter-02 fix location (tool catalog / prompt / DAG / retries).

## Where this hands off

- `scorer.py` is the exact shape exercise-01 asks you to productionise — tests, CLI, equivalence classes, a Level-4 AST slot. The surgical-edit fixtures above become its table-driven test cases.
- The verdict JSON this lab emits is the input to exercise-02's `roll_up` function; the summary's `flagged_verdicts` is the list chapter 03 extends with final-answer and budget verdicts before applying the rubric's floors.
- Exercise-03 wraps a production version of this scorer inside an Inspect `Scorer`; `state.messages` (plus `state.metadata["reference_trajectory"]`) stands in for the JSON files you loaded here.
- Lab-02 pairs the per-step vocabulary with the chapter-04 budget vocabulary; lab-03 closes the triad on replay.
