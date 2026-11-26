# lab-02: Compute `budget_credit` by Hand, Then by Code, Then Shift the Attribution

**Estimated duration:** 45–60 minutes
**Pairs with:** chapter 04 of this module.

## Objective

Make chapter 04's budget arithmetic physical. You compute `over_fraction`, per-family `credit`, and the `min`-across-families `budget_credit` on paper for the chapter-01 trajectory; then you check the same numbers with a short script; then you make targeted edits to the trajectory and watch the attribution block (`cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool`) shift to point at the span your edit made worst. The intent is to internalise *why the attribution block is load-bearing*: a rollout gate that fires "cost over budget" without a dominant span is a bug report with no stack trace (chapter 04).

At the end, the words "p95 cost regressed, dominated by `llm.plan.3`" produce a mental picture of which arithmetic ran and which span it pointed at.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-01 builds the full budget scorer as a deliverable (CLI + tests + attribution + price-table versioning).
- Exercise-02 rolls budget credits into the chapter-03 rubric.
- Exercise-03 runs the budget scorer inside Inspect.

This lab is pencil-and-paper arithmetic made reproducible in Python, with surgical edits that move the attribution.

## Prerequisites

- Chapter 04 read, plus chapter 01 for the running example.
- Python 3.10+ on `$PATH`. Stdlib only.
- A pencil and paper (seriously — step 2 is faster on paper than in the shell).

## Layout you will build

```
mod-103-lab-02/
  fixtures/
    slo.json                # ceilings for the task family
    price_table.json        # per-model input / output / cached prices
    trajectory.json         # the chapter-01 bad trajectory, canned
  budget.py                 # the budget scorer
  edit.py                   # helper that applies one named edit at a time
```

---

## Steps

### Step 1 — Create the working directory and fixtures

```bash
mkdir -p mod-103-lab-02/fixtures && cd mod-103-lab-02
```

Create `fixtures/slo.json` — the SLO ceilings from chapter 01's illustrative ledger:

```json
{
  "task_family": "order_status_not_shipped_policy",
  "cost_usd_ceiling": 0.020,
  "time_limit_s":     8,
  "max_steps":        5,
  "max_tool_calls":   4,
  "max_retries_per_tool": 1,
  "price_table_version": "v12-lab02"
}
```

Create `fixtures/price_table.json` — illustrative prices per 1k tokens:

```json
{
  "version": "v12-lab02",
  "models": {
    "anthropic/claude-opus-4-7": {
      "input_per_1k":  0.0150,
      "output_per_1k": 0.0750,
      "cached_input_per_1k": 0.0015
    }
  }
}
```

Create `fixtures/trajectory.json` — the chapter-01 bad trajectory, canned with the per-span cost and duration numbers from the chapter's worked example. Cost figures here are illustrative and keep the chapter's shape; precise dollars depend on the price table.

```json
{
  "trajectory_id": "c31f7a-lab02",
  "task_family":   "order_status_not_shipped_policy",
  "agent": {
    "span_id":  "s_root",
    "start_ms": 0,
    "end_ms":   14200,
    "app.cost_usd": 0.219
  },
  "steps": [
    {"step_id": 1, "kind": "LLM",  "span_id": "llm.plan",
     "model": "anthropic/claude-opus-4-7",
     "tokens": {"input": 652, "output": 224},
     "cost_usd": 0.0009, "duration_ms": 500, "start_ms":   0, "end_ms":  500},
    {"step_id": 2, "kind": "TOOL", "span_id": "tool.sql_query",
     "tool_name": "sql_query",
     "arguments": {"query": "SELECT * FROM orders WHERE id=12345"},
     "duration_ms": 712, "start_ms":  500, "end_ms": 1212, "status": "ok"},
    {"step_id": 3, "kind": "LLM",  "span_id": "llm.plan.2",
     "model": "anthropic/claude-opus-4-7",
     "tokens": {"input": 712, "output": 209},
     "cost_usd": 0.0011, "duration_ms": 600, "start_ms": 1212, "end_ms": 1812},
    {"step_id": 4, "kind": "TOOL", "span_id": "retriever.docs.search",
     "tool_name": "docs.search",
     "arguments": {"query": "order 12345 status", "top_k": 3},
     "duration_ms": 400, "start_ms": 1812, "end_ms": 2212, "status": "ok"},
    {"step_id": 5, "kind": "TOOL", "span_id": "retriever.docs.search.2",
     "tool_name": "docs.search",
     "arguments": {"query": "order 12345 shipping", "top_k": 3},
     "duration_ms": 400, "start_ms": 2212, "end_ms": 2612, "status": "ok"},
    {"step_id": 6, "kind": "LLM",  "span_id": "llm.plan.3",
     "model": "anthropic/claude-opus-4-7",
     "tokens": {"input": 904, "output": 200},
     "cost_usd": 0.0014, "duration_ms": 700, "start_ms": 2612, "end_ms": 3312},
    {"step_id": 7, "kind": "TOOL", "span_id": "tool.crm_lookup",
     "tool_name": "crm.lookup",
     "arguments": {"order_id": 12345},
     "duration_ms": 2103, "start_ms": 3312, "end_ms":  5415, "status": "error"},
    {"step_id": 8, "kind": "TOOL", "span_id": "tool.crm_lookup.retry.1",
     "tool_name": "crm.lookup",
     "arguments": {"order_id": 12345},
     "duration_ms": 2008, "start_ms": 5415, "end_ms":  7423, "status": "error"},
    {"step_id": 9, "kind": "TOOL", "span_id": "tool.crm_lookup.retry.2",
     "tool_name": "crm.lookup",
     "arguments": {"order_id": 12345},
     "duration_ms": 1912, "start_ms": 7423, "end_ms":  9335, "status": "ok"},
    {"step_id": 10, "kind": "LLM", "span_id": "llm.plan.4",
     "model": "anthropic/claude-opus-4-7",
     "tokens": {"input": 1088, "output": 200},
     "cost_usd": 0.0018, "duration_ms": 800, "start_ms": 9335, "end_ms": 10135},
    {"step_id": 11, "kind": "TOOL", "span_id": "tool.open_ticket",
     "tool_name": "open_ticket",
     "arguments": {"order_id": 12345, "reason": "no_shipping_update"},
     "duration_ms": 600, "start_ms": 10135, "end_ms": 10735, "status": "ok"},
    {"step_id": 12, "kind": "LLM", "span_id": "llm.reply",
     "model": "anthropic/claude-opus-4-7",
     "tokens": {"input": 1221, "output": 200},
     "cost_usd": 0.0021, "duration_ms": 900, "start_ms": 10735, "end_ms": 11635}
  ]
}
```

### Step 2 — Compute `budget_credit` on paper

Fill the table below using only chapter 04's arithmetic. Keep your sheet; Step 3's script output should match within rounding.

| Family | Observed | Ceiling | `over_fraction = max(0, obs/ceil − 1)` | `credit = clamp(1 − over_fraction, 0, 1)` |
|---|---|---|---|---|
| Cost (USD) | `0.219` | `0.020` | ___ | ___ |
| Latency (s, wall-clock) | `end_ms − start_ms` on the root = ___ s | `8` | ___ | ___ |
| Step count (one iteration per LLM + its tools) | ___ | `5` | ___ | ___ |
| Tool calls | ___ (count collapsed retries as **one** tool call) | `4` | ___ | ___ |
| Max retries per tool | ___ | `1` | ___ | ___ |

Then compute `budget_credit = min(credit over families)`. Pay attention to chapter 04's rule: any family at ≥ 2× its ceiling pins the family credit at `0`.

**Expected answers (sanity check):**

- `cost`: with the lab's `price_table.json` (`input_per_1k = 0.0150`, `output_per_1k = 0.0750`) the scorer re-computes cost from tokens and gets `cost_observed ≈ 0.146` (sum across five LLMs — the exact figure depends on the token counts in the fixture). `over_fraction = 0.146 / 0.020 − 1 ≈ 6.3`; `credit = 0` (any family at ≥ 2× its ceiling pins to 0). Chapter 04 is explicit: "do not store a flat `cost_per_token`", so the stored `cost_usd` on each step is **not** what the scorer reads — it reads tokens × versioned price table.
- `latency`: wall-clock = `(agent.end_ms − agent.start_ms) / 1000 = 14.2 s`; over_fraction = `14.2 / 8 − 1 = 0.775`; credit = `0.225`. (Wall-clock is the AGENT span, *not* the sum of child durations — chapter 04's "Latency arithmetic for parallel tool calls".)
- `steps`: count the agent-loop iterations — one LLM span plus its downstream tools is one step. In this trajectory the LLMs are `llm.plan`, `llm.plan.2`, `llm.plan.3`, `llm.plan.4`, `llm.reply` — **5 steps**. Over_fraction = `0`; credit = `1.0`.
  - Chapter 04 is explicit: "a step with parallel tool calls is one step, not N". Our trajectory has no parallel calls, but the retry chain under `crm.lookup` is one step, not three.
- `tool_calls`: collapsed retries make the call count: `sql_query`, `docs.search`, `docs.search.2`, `crm.lookup` (with retry_count=2), `open_ticket` — **5 tool calls**. Over_fraction = `5/4 − 1 = 0.25`; credit = `0.75`.
- `retries`: `max_retries_per_tool observed = 2` (on `crm.lookup`); ceiling = `1`. Over_fraction = `1.0`; credit = `0`.
- `budget_credit = min(0, 0.225, 1.0, 0.75, 0) = 0`.

If your paper disagrees with any of these, re-read chapter 04's "Step count, tool count, retry count" definitions before moving on. The common slip is treating each retry as its own tool call; chapter 04 is explicit that a collapsed retry is **one** tool call plus a retry count.

### Step 3 — Write the budget scorer (`budget.py`)

The scorer should take `(trajectory, slo, price_table)` and emit the chapter-04 verdict JSON — per-family credits, flagged verdicts, and the attribution block. Keep it stdlib-only.

```python
# budget.py
"""Chapter-04 budget scorer. No model calls; stdlib only.

Four families (cost, latency, step-count, tool-call-count, retry), one
family-credit computation each, a worst-family-wins roll-up, and an
attribution block pointing at the single span that drove each breach."""

import json
import sys
from pathlib import Path


def _linear_credit(observed: float, ceiling: float) -> tuple[float, float]:
    """Return (over_fraction, credit) per chapter 04."""
    if ceiling <= 0:
        raise ValueError("ceiling must be > 0")
    over = max(0.0, observed / ceiling - 1.0)
    credit = max(0.0, min(1.0, 1.0 - over))
    # Chapter 04: pin to 0 at 2× over the ceiling.
    if observed >= 2 * ceiling:
        credit = 0.0
    return over, credit


def _cost_per_step(step: dict, price_table: dict) -> float:
    """Use the response model's input / output rates from the price table.
    The scorer never infers a cost; it charges against a versioned table."""
    if step["kind"] != "LLM":
        return 0.0
    rates = price_table["models"][step["model"]]
    input_cost  = step["tokens"]["input"]  / 1000.0 * rates["input_per_1k"]
    output_cost = step["tokens"]["output"] / 1000.0 * rates["output_per_1k"]
    return round(input_cost + output_cost, 6)


def _collapse_tools(steps: list) -> list:
    """Chapter 02: consecutive identical (tool_name, arguments) calls that
    differ only in success status collapse to one. Retry count is retained."""
    out = []
    for s in steps:
        if s["kind"] != "TOOL":
            out.append({**s, "retry_count": 0})
            continue
        key = (s["tool_name"], json.dumps(s["arguments"], sort_keys=True))
        if out and out[-1].get("_key") == key:
            out[-1]["retry_count"] += 1
            # Keep the latest (successful) span_id and sum durations so the
            # latency attribution still points at the retry-storm cluster.
            out[-1]["duration_ms"] += s["duration_ms"]
            out[-1]["span_id"] = s["span_id"]
            out[-1]["status"]  = s["status"]
            continue
        out.append({**s, "_key": key, "retry_count": 0})
    return out


def score(trajectory: dict, slo: dict, price_table: dict) -> dict:
    steps = trajectory["steps"]
    collapsed = _collapse_tools(steps)

    # cost: sum across LLM spans using the price table.
    per_step_cost = [{"span_id": s["span_id"], "cost": _cost_per_step(s, price_table)}
                     for s in collapsed if s["kind"] == "LLM"]
    cost_observed = round(sum(x["cost"] for x in per_step_cost), 6)

    # latency: wall-clock on the AGENT span (not sum-of-steps).
    agent = trajectory["agent"]
    lat_wall_s = (agent["end_ms"] - agent["start_ms"]) / 1000.0

    # steps: one per LLM (chapter 04: parallel tools do not add steps).
    step_count = sum(1 for s in collapsed if s["kind"] == "LLM")

    # tool_calls: one per collapsed tool entry.
    tool_calls = sum(1 for s in collapsed if s["kind"] == "TOOL")

    # retries: max retry_count across tools; also track per-tool.
    retries_per_tool = {}
    for s in collapsed:
        if s["kind"] != "TOOL":
            continue
        retries_per_tool[s["tool_name"]] = max(
            retries_per_tool.get(s["tool_name"], 0), s["retry_count"]
        )
    max_retries = max(retries_per_tool.values(), default=0)

    # Per-family credit.
    cost_over,   cost_credit   = _linear_credit(cost_observed, slo["cost_usd_ceiling"])
    lat_over,    lat_credit    = _linear_credit(lat_wall_s,    slo["time_limit_s"])
    steps_over,  steps_credit  = _linear_credit(step_count,    slo["max_steps"])
    tools_over,  tools_credit  = _linear_credit(tool_calls,    slo["max_tool_calls"])
    retry_over,  retry_credit  = _linear_credit(max_retries,   slo["max_retries_per_tool"])

    family_credits = {
        "cost": cost_credit, "latency": lat_credit,
        "steps": steps_credit, "tool_calls": tools_credit,
        "retries": retry_credit,
    }
    budget_credit = min(family_credits.values())

    # Attribution: single span per family per chapter 04.
    # Latency attribution uses the raw per-span duration, NOT the collapsed
    # retry-cluster duration. Chapter 04 is explicit: "the slowest span on
    # the critical path", which is one physical span.
    cost_dom = max(per_step_cost, key=lambda x: x["cost"])["span_id"] if per_step_cost else None
    lat_dom  = max(steps, key=lambda s: s.get("duration_ms", 0))["span_id"]
    retry_dom = None
    if retries_per_tool:
        tool, n = max(retries_per_tool.items(), key=lambda kv: kv[1])
        retry_dom = tool if n > 0 else None

    flagged = []
    if cost_credit  < 1.0: flagged.append("cost_over_budget")
    if lat_credit   < 1.0: flagged.append("latency_over_budget")
    if steps_credit < 1.0: flagged.append("steps_over_budget")
    if tools_credit < 1.0: flagged.append("tool_calls_over_budget")
    if retry_credit < 1.0: flagged.append("retry_budget_blown")

    return {
        "trajectory_id":  trajectory["trajectory_id"],
        "task_family":    trajectory["task_family"],
        "price_table_version": price_table["version"],
        "budget_credit":  round(budget_credit, 3),
        "budget_axes": {
            "cost":        {"observed": cost_observed,         "ceiling": slo["cost_usd_ceiling"],
                            "over_fraction": round(cost_over, 3), "credit": round(cost_credit, 3)},
            "latency":     {"observed_wallclock_s": lat_wall_s, "ceiling_s": slo["time_limit_s"],
                            "over_fraction": round(lat_over, 3),  "credit": round(lat_credit, 3)},
            "steps":       {"observed": step_count,             "ceiling": slo["max_steps"],
                            "over_fraction": round(steps_over, 3), "credit": round(steps_credit, 3)},
            "tool_calls":  {"observed": tool_calls,             "ceiling": slo["max_tool_calls"],
                            "over_fraction": round(tools_over, 3), "credit": round(tools_credit, 3)},
            "retries":     {"observed_max": max_retries,        "ceiling_max": slo["max_retries_per_tool"],
                            "over_fraction": round(retry_over, 3), "credit": round(retry_credit, 3),
                            "per_tool": retries_per_tool},
        },
        "attribution": {
            "cost_dominant_span":    cost_dom,
            "latency_dominant_span": lat_dom,
            "retry_dominant_tool":   retry_dom,
        },
        "flagged_verdicts": flagged,
    }


if __name__ == "__main__":
    traj  = json.loads(Path(sys.argv[1]).read_text())
    slo   = json.loads(Path(sys.argv[2]).read_text())
    price = json.loads(Path(sys.argv[3]).read_text())
    print(json.dumps(score(traj, slo, price), indent=2))
```

### Step 4 — Run the scorer and compare to your paper

```bash
python3 budget.py fixtures/trajectory.json fixtures/slo.json fixtures/price_table.json
```

The key fields to look for:

- `budget_credit` is `0.0` (a `min` across families; cost pinned at 0; retries pinned at 0).
- `budget_axes.cost.over_fraction` ≈ `6.3` on the lab's price table, `credit` = `0.0`.
- `budget_axes.latency.observed_wallclock_s` = `14.2` (from the AGENT span, not the sum of child durations).
- `budget_axes.steps.observed` = `5` (**five LLMs**, not twelve items — chapter 04 counts iterations, not spans).
- `budget_axes.tool_calls.observed` = `5` (retries collapsed to one tool call).
- `budget_axes.retries.observed_max` = `2` on `crm.lookup`.
- `attribution.cost_dominant_span` = `llm.reply` (highest single-span cost at 1221-in / 200-out tokens — the model's output tokens cost 5× its input tokens in the lab price table). Chapter 04 is explicit: the dominant span is the single biggest contributor; for the running example in chapter 01 the illustration was `llm.plan.3`, but the exact winner depends on the price table and the token counts in your canned fixture.
- `attribution.latency_dominant_span` = `tool.crm_lookup` (the first retry at 2103 ms is the single slowest span on the critical path — chapter 04 attributes by individual span duration, not by cluster duration).
- `attribution.retry_dominant_tool` = `crm.lookup`.

Chapter 04's point: `budget_credit=0` + three attributions = a stack-trace-shaped bug report. A rollout gate ought to deep-link into each dominant span before paging anyone.

### Step 5 — Surgical edit #1: shift the cost attribution

Increase the output-token count of `llm.plan.4` from `200` to `1200` (a prompt refactor that silently blew up the plan's reasoning length). Compute by hand which span now has the highest cost, then confirm with the scorer.

```bash
python3 - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("fixtures/trajectory.json").read_text())
t["trajectory_id"] = "c31f7a-edit1"
for s in t["steps"]:
    if s["span_id"] == "llm.plan.4":
        s["tokens"]["output"] = 1200
        s["cost_usd"] = round(s["tokens"]["input"] / 1000 * 0.015 +
                              s["tokens"]["output"] / 1000 * 0.075, 6)
Path("fixtures/trajectory_cost_shift.json").write_text(json.dumps(t, indent=2))
PY
python3 budget.py fixtures/trajectory_cost_shift.json fixtures/slo.json fixtures/price_table.json | python3 -m json.tool | grep -E '(cost_dominant_span|cost_over_budget)'
```

**Expected:** `cost_dominant_span` is now `llm.plan.4` (highest single-span cost); `cost_over_budget` is still in `flagged_verdicts`; `budget_credit` is still `0.0` (retries still pin it). The attribution shifts even though the credit does not.

This is the pattern a real on-call follows: the pass/fail boolean is already clear from the floor trip (chapter 03), but the *where to look* field is what the dashboard displays and what drives the fix.

### Step 6 — Surgical edit #2: shift the latency attribution

Tighten the `crm.lookup` retry chain to a single successful call (as if the dependency recovered) and simultaneously stretch `tool.open_ticket` from `600 ms` to `6000 ms`. The latency attribution should now point at `tool.open_ticket`.

```bash
python3 - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("fixtures/trajectory.json").read_text())
t["trajectory_id"] = "c31f7a-edit2"
# Remove the two failed crm.lookup entries; keep only the successful one.
t["steps"] = [s for s in t["steps"]
              if s["span_id"] not in {"tool.crm_lookup", "tool.crm_lookup.retry.1"}]
# Rename the surviving retry to the canonical crm.lookup span_id.
for s in t["steps"]:
    if s["span_id"] == "tool.crm_lookup.retry.2":
        s["span_id"] = "tool.crm_lookup"
    if s["span_id"] == "tool.open_ticket":
        s["duration_ms"] = 6000
Path("fixtures/trajectory_latency_shift.json").write_text(json.dumps(t, indent=2))
PY
python3 budget.py fixtures/trajectory_latency_shift.json fixtures/slo.json fixtures/price_table.json | python3 -m json.tool | grep -E '(latency_dominant_span|retry_dominant_tool|retries)'
```

**Expected:** `latency_dominant_span` is now `tool.open_ticket`; `retry_dominant_tool` is `null` (no retries left; the scorer returns `None` when `max_retries_observed == 0`); `retry_budget_blown` is no longer in `flagged_verdicts`. The wall-clock **latency may still be over budget**, but the attribution routes the on-call to `open_ticket` (the new worst span), not to `crm.lookup`.

Chapter 04's lesson: attribution follows the metric, not the symptom. Fix the right span.

### Step 7 — Surgical edit #3: clear the floor-level breaches entirely

Reduce `llm.reply` tokens drastically, keep the single-call `crm.lookup`, and drop `docs.search.2` (the redundant retrieval). The trajectory should now be inside every ceiling and the scorer should emit `budget_credit = 1.0` with an empty `flagged_verdicts` list.

```bash
python3 - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("fixtures/trajectory.json").read_text())
t["trajectory_id"] = "c31f7a-happy"
t["steps"] = [s for s in t["steps"]
              if s["span_id"] not in {"tool.crm_lookup", "tool.crm_lookup.retry.1",
                                      "retriever.docs.search.2"}]
for s in t["steps"]:
    if s["span_id"] == "tool.crm_lookup.retry.2":
        s["span_id"] = "tool.crm_lookup"
    if s["kind"] == "LLM":
        # Compress every LLM to 100 in / 20 out — cheap and fast.
        s["tokens"] = {"input": 100, "output": 20}
        s["cost_usd"] = round(100/1000*0.015 + 20/1000*0.075, 6)
        s["duration_ms"] = 100
    else:
        # Compress every TOOL to a 100ms stub.
        s["duration_ms"] = 100
# Re-stitch the AGENT wall-clock from the compacted steps.
t["agent"]["start_ms"] = 0
# Lay the steps out end-to-end for a clean wall-clock number.
cursor = 0
for s in t["steps"]:
    s["start_ms"] = cursor
    cursor += s["duration_ms"]
    s["end_ms"] = cursor
t["agent"]["end_ms"] = cursor
Path("fixtures/trajectory_happy.json").write_text(json.dumps(t, indent=2))
PY
python3 budget.py fixtures/trajectory_happy.json fixtures/slo.json fixtures/price_table.json | python3 -m json.tool | grep -E '(budget_credit|flagged_verdicts)'
```

**Expected:** `"budget_credit": 1.0` and `"flagged_verdicts": []`. Every family credit is `1.0`; nothing crosses a ceiling.

### Step 8 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] Your paper table in Step 2 matches the scorer's `budget_axes` output within rounding.
- [ ] Step count is `5` (one iteration per LLM span), not `12` (one per child of the agent); you can explain why.
- [ ] Tool-call count is `5` (collapsed retry on `crm.lookup`), not `6`; the retry count on `crm.lookup` is `2`.
- [ ] The three attribution fields are populated on every non-trivial fixture and point at the span your edit made worst.
- [ ] The happy-path fixture emits `budget_credit = 1.0` and `flagged_verdicts = []`.
- [ ] You can state, in one sentence per family, why chapter 04 uses `min` across families instead of a weighted sum.

### Step 9 — Stretch: shadow a rollout gate

Chapter 04's offline gate reads `cost_p95 ≤ cost_ceiling * (1 + tolerance)` across a suite. Take the four trajectories you now have (`trajectory.json`, `_cost_shift.json`, `_latency_shift.json`, `_happy.json`), score each, and compute the p95 of `cost_observed`. Does the suite pass under the chapter-04 default tolerance of `0.1`?

```bash
python3 - <<'PY'
import json, statistics
from pathlib import Path
from budget import score

slo = json.loads(Path("fixtures/slo.json").read_text())
price = json.loads(Path("fixtures/price_table.json").read_text())
paths = ["fixtures/trajectory.json", "fixtures/trajectory_cost_shift.json",
         "fixtures/trajectory_latency_shift.json", "fixtures/trajectory_happy.json"]
costs = []
for p in paths:
    v = score(json.loads(Path(p).read_text()), slo, price)
    costs.append(v["budget_axes"]["cost"]["observed"])
    print(f"{Path(p).name}: cost={v['budget_axes']['cost']['observed']:.4f}")
# nearest-method quantile; four points so p95 is dominated by the max.
cost_p95 = statistics.quantiles(costs, n=20, method="exclusive")[-1]
gate = slo["cost_usd_ceiling"] * 1.1
print(f"cost_p95={cost_p95:.4f}  gate={gate:.4f}  pass={cost_p95 <= gate}")
PY
```

**Expected:** with the chapter-01 bad trajectory in the suite, `cost_p95` is dominated by its `0.219` and the gate fails. The chapter-04 point lands: the suite's p95 (not its mean) is what the gate reads; one tail trajectory is enough to block the merge.

## Where this hands off

- The verdict shape above is the `BudgetVerdict` exercise-01 asks you to productionise — tests on retry collapsing, parallel-tool steps, and the price-table discipline.
- The scalar `budget_credit` flows into exercise-02's `roll_up` as one of the six rubric axes; the `flagged_verdicts` list extends the chapter-02 vocabulary chapter 03 rolls into the partial-credit rubric.
- The attribution block (`cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool`) is what mod-107's online dashboard groups trajectories by; a Pareto chart of `cost_dominant_span` across a traffic sample is the artefact chapter 04 points at.
- Lab-01 covered the per-step axis; lab-03 closes the triad by making the failing trajectory from this lab deterministically replayable.
