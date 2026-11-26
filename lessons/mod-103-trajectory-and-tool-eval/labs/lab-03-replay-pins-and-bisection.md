# lab-03: Materialise the Six Replay Pins, Then Bisect One at a Time

**Estimated duration:** 60–90 minutes
**Pairs with:** chapter 07 of this module.

## Objective

Make chapter 07's replay contract mechanical. You take one failing trajectory, materialise the six pins (model, sampling, messages, tool returns, environment, harness), build the recorded tool-response table, run a tiny "replay runner" that reproduces the trajectory *without calling a real model or a real tool*, confirm that the scorer from lab-01 emits the same `flagged_verdicts` on the replay, and then walk one deterministic bisection — change exactly one pin, replay, watch exactly one verdict clear.

At the end you have an artefact that converts "the agent is broken, let me re-run it and hope for the best" into "I changed pin X; the retry verdict cleared; the cost verdict did not; the fix is in pin X".

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-03 wires an Inspect Task against a real model.
- Exercise-05 walks the full six-step post-mortem (lift / pin / reproduce / bisect / fix / learn) against a trace your Inspect run produced, with the real model called at replay time.

This lab is **stdlib-only**: no Inspect, no model API key, no network. The replay runner stubs the model with a captured message-sequence table so the trajectory is reproducible on a plane. The point is to build an intuition for what each pin is actually *for* — not to run a production replay.

## Prerequisites

- Chapter 07 read. Chapters 02 and 04 helpful (the scorer and verdict vocabulary are the same ones lab-01 and lab-02 covered).
- Python 3.10+ on `$PATH`. Stdlib only.
- The scorer from lab-01 (`scorer.py`) and the reference JSON (`fixtures/reference.json`). If you have not done lab-01, copy both into this lab's directory — they are small.
- Optional: the budget scorer from lab-02 (`budget.py`), to check that the budget verdict also reproduces.

## Layout you will build

```
mod-103-lab-03/
  fixtures/
    reference.json             # from lab-01
    original_trajectory.json   # the failing run, as captured
  pins/
    replay_context.json        # the six pins, materialised
  recorded/
    tool_responses.json        # the recorded tool-response table
    captured_model_calls.json  # per-step model inputs + outputs (stubs the model)
  replay.py                    # the replay runner
  bisect.py                    # helper that edits one pin and re-runs
  scorer.py                    # from lab-01 (or vendor it)
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-103-lab-03/{fixtures,pins,recorded} && cd mod-103-lab-03
```

### Step 2 — Vendor the lab-01 scorer and reference

Copy `scorer.py` and `fixtures/reference.json` from your lab-01 directory into this one. If you did not keep lab-01 around, re-author them from lab-01 Steps 2 and 4 before continuing.

```bash
cp ../mod-103-lab-01/scorer.py .
cp ../mod-103-lab-01/fixtures/reference.json fixtures/
```

**Sanity-check:** `python3 -c "from scorer import score; print('ok')"` prints `ok`.

### Step 3 — Author the original (failing) trajectory

This is the failing trajectory you will replay. The failure mode is chapter 01's golden shortcut *plus* a retry storm — so the scorer should flag `unauthorised_tool` (sql_query was picked) and `missing_required_tool` (docs.search was never called before the reply fired). Save as `fixtures/original_trajectory.json`:

```json
{
  "trajectory_id": "traj-replay-source",
  "task_family":   "order_status_not_shipped_policy",
  "steps": [
    {"step_id": 1, "span_id": "llm.plan",  "kind": "LLM",
     "model": "anthropic/claude-opus-4-7",
     "messages_in":  [{"role": "system", "content": "support planner v3 (ambiguous)"},
                      {"role": "user",   "content": "Where is my order 12345?"}],
     "messages_out": [{"role": "assistant",
                       "tool_calls": [{"name": "sql_query",
                                       "args": {"query": "SELECT * FROM orders WHERE id=12345"}}]}]},
    {"step_id": 2, "span_id": "tool.sql_query", "kind": "TOOL",
     "tool_name": "sql_query",
     "arguments": {"query": "SELECT * FROM orders WHERE id=12345"}},
    {"step_id": 3, "span_id": "llm.plan.2", "kind": "LLM",
     "model": "anthropic/claude-opus-4-7",
     "messages_in":  [{"role": "system", "content": "support planner v3 (ambiguous)"},
                      {"role": "user",   "content": "Where is my order 12345?"},
                      {"role": "assistant", "tool_calls": [{"name": "sql_query",
                                              "args": {"query": "SELECT * FROM orders WHERE id=12345"}}]},
                      {"role": "tool",       "content": "{\"id\": 12345, \"status\": \"in_transit\"}"}],
     "messages_out": [{"role": "assistant",
                       "tool_calls": [{"name": "crm.lookup", "args": {"order_id": 12345}}]}]},
    {"step_id": 4, "span_id": "tool.crm_lookup",         "kind": "TOOL",
     "tool_name": "crm.lookup", "arguments": {"order_id": 12345}},
    {"step_id": 5, "span_id": "tool.crm_lookup.retry.1", "kind": "TOOL",
     "tool_name": "crm.lookup", "arguments": {"order_id": 12345}},
    {"step_id": 6, "span_id": "tool.crm_lookup.retry.2", "kind": "TOOL",
     "tool_name": "crm.lookup", "arguments": {"order_id": 12345}},
    {"step_id": 7, "span_id": "llm.reply", "kind": "LLM",
     "model": "anthropic/claude-opus-4-7",
     "messages_in":  [{"role": "assistant", "content": "... (truncated for the lab)"}],
     "messages_out": [{"role": "assistant",
                       "content": "Your order 12345 is in transit; ETA in 2 days."}]}
  ]
}
```

Score it now to confirm the failure fires:

```bash
python3 -c "
import json
from pathlib import Path
from scorer import score
traj = json.loads(Path('fixtures/original_trajectory.json').read_text())
ref  = json.loads(Path('fixtures/reference.json').read_text())
print(score(traj, ref)['summary']['flagged_verdicts'])"
```

**Expected:** `['missing_required_tool', 'unauthorised_tool']` (in sorted order). Keep this list in front of you; the replay must reproduce it exactly. (`ordering_violation` does **not** fire here — `open_ticket` is never called, so no DAG edge is violated; the trajectory simply bypasses the required retriever. Chapter 02 is explicit: missing_required_tool and ordering_violation are distinct verdicts with distinct attribution.)

### Step 4 — Materialise the six pins (`pins/replay_context.json`)

Chapter 07 lists six pin families: model identity, sampling parameters, exact message list sent to the model at each step, tool returns, environment, and harness version. Author them as a single JSON document. Hash the serialised message list so the replay runner can sanity-check it did not drift.

```python
# One-shot script to build pins/replay_context.json. Keep the output;
# the replay runner reads it verbatim.
python3 - <<'PY'
import hashlib, json
from pathlib import Path

traj = json.loads(Path("fixtures/original_trajectory.json").read_text())

# Build the message-list hash: concatenate every LLM step's messages_in
# in order and hash the serialisation. This is the pin for "exact
# message list sent to the model at each step".
msg_bytes = b""
for s in traj["steps"]:
    if s["kind"] == "LLM":
        msg_bytes += json.dumps(s["messages_in"], sort_keys=True).encode()
msg_hash = "sha256:" + hashlib.sha256(msg_bytes).hexdigest()

pins = {
    "trajectory_id": traj["trajectory_id"],
    "model": {
        "response_model_id": "anthropic/claude-opus-4-7",
        "request_model_id":  "anthropic/claude-opus-4-7"
    },
    "sampling": {
        "temperature": 0.2, "top_p": 0.9, "max_tokens": 512, "seed": None
    },
    "messages_hash": msg_hash,
    "tool_response_table": "recorded/tool_responses.json",
    "captured_model_calls": "recorded/captured_model_calls.json",
    "environment": {
        "clock_at_record_time": "2026-10-01T14:32:08Z",
        "feature_flags":        {"fast_path": "off"},
        "locale":               "en-US",
        "agent_id":             "support-agent-prod-5"
    },
    "harness": {
        "inspect_version": "x.y.z",   # pin in production; placeholder here
        "python_version":  "3.11.9",
        "lab_mode":        True
    }
}
Path("pins/replay_context.json").write_text(json.dumps(pins, indent=2))
print(json.dumps(pins, indent=2))
PY
```

**Expected:** `pins/replay_context.json` is written and the six top-level keys match chapter 07's list.

### Step 5 — Build the recorded tool-response table

Chapter 07 is explicit: record the **full retry sequence** for retried tools, and canonicalise arguments before indexing. Side-effectful tools are never executed on replay. Build `recorded/tool_responses.json`:

```json
[
  {"step": 2, "tool": "sql_query",
   "args": {"query": "SELECT * FROM orders WHERE id=12345"},
   "return": {"id": 12345, "status": "in_transit", "eta_days": 2},
   "latency_ms": 712, "status": "ok", "replay_mode": "stub_only"},
  {"step": 4, "tool": "crm.lookup",
   "args": {"order_id": 12345},
   "return_sequence": [
     {"status": "error", "code": 502, "latency_ms": 2103},
     {"status": "error", "code": 502, "latency_ms": 2008},
     {"status": "ok",    "payload": {"shipment_id": "SHP-0098"}, "latency_ms": 1912}
   ],
   "replay_mode": "stub_only"}
]
```

Three rules chapter 07 states — enforce them in your table-builder, by eye for now, in code in exercise-05:

- **Full retry sequence**, not just the final success.
- **Canonicalise arguments** (sort keys, coerce numeric strings).
- **Side-effectful tools** (`send_email`, in your real surface `open_ticket`) are flagged `replay_mode: "stub_only"`: the replay harness **never** executes them.

Similarly, record every model call's inputs + outputs so the replay runner can stub out the model deterministically. Save as `recorded/captured_model_calls.json`:

```python
python3 - <<'PY'
import json
from pathlib import Path

traj = json.loads(Path("fixtures/original_trajectory.json").read_text())
out = []
for s in traj["steps"]:
    if s["kind"] != "LLM":
        continue
    out.append({
        "step_id":     s["step_id"],
        "model":       s["model"],
        "messages_in": s["messages_in"],
        "output":      s["messages_out"],
    })
Path("recorded/captured_model_calls.json").write_text(json.dumps(out, indent=2))
print(f"{len(out)} model calls captured")
PY
```

**Expected:** three model calls (plan, plan.2, reply) captured.

### Step 6 — Write the replay runner (`replay.py`)

The replay runner takes `pins/replay_context.json` + the two recorded tables + the original trajectory, and produces a new trajectory object that is byte-identical to the original except for the `trajectory_id`. **It calls no model, it executes no tool.** That is the whole point.

```python
# replay.py
"""Deterministic replay runner. No model calls; no tool execution.

Reads pins/replay_context.json, recorded/tool_responses.json, and
recorded/captured_model_calls.json. For each LLM step, yields the
captured output. For each TOOL step, yields the recorded return (or
the next entry of the retry sequence). The output is a trajectory
object the lab-01 scorer can read."""

import copy
import hashlib
import json
import sys
from pathlib import Path


def _canon_args(args: dict) -> str:
    return json.dumps(args, sort_keys=True)


def _hash_msgs(steps: list) -> str:
    buf = b""
    for s in steps:
        if s["kind"] == "LLM":
            buf += json.dumps(s["messages_in"], sort_keys=True).encode()
    return "sha256:" + hashlib.sha256(buf).hexdigest()


def replay(pins_path: Path, original_path: Path) -> dict:
    pins = json.loads(pins_path.read_text())
    original = json.loads(original_path.read_text())
    tool_table_path = Path(pins["tool_response_table"])
    model_calls_path = Path(pins["captured_model_calls"])
    tool_entries = json.loads(tool_table_path.read_text())
    model_calls = json.loads(model_calls_path.read_text())

    # Pin check: the serialised message list from `original` must hash to
    # the value stored in pins. Fail-closed — this is the "exact message
    # list" pin chapter 07 names.
    observed_hash = _hash_msgs(original["steps"])
    if observed_hash != pins["messages_hash"]:
        raise SystemExit(
            f"messages_hash mismatch: pin={pins['messages_hash']} observed={observed_hash}"
        )

    # Index model calls by step_id and tool returns by (tool_name, canon args).
    model_by_step = {m["step_id"]: m for m in model_calls}
    tool_by_key   = {}
    retry_cursors = {}
    for t in tool_entries:
        key = (t["tool"], _canon_args(t["args"]))
        tool_by_key[key] = t
        retry_cursors[key] = 0

    replayed_steps = []
    for s in original["steps"]:
        s = copy.deepcopy(s)
        if s["kind"] == "LLM":
            m = model_by_step[s["step_id"]]
            # Replace the step's output with the captured one. In a real
            # replay, this is where the model would be called; here we
            # just hand back what the recording said the model emitted.
            s["messages_out"] = m["output"]
            replayed_steps.append(s)
        elif s["kind"] == "TOOL":
            key = (s["tool_name"], _canon_args(s["arguments"]))
            entry = tool_by_key.get(key)
            if entry is None:
                # Chapter 07: interception must be total. Unknown key = fail.
                raise SystemExit(
                    f"no recorded tool response for step {s['step_id']}: "
                    f"tool={s['tool_name']} args={s['arguments']}"
                )
            # Side-effectful tools must be stub_only; the lab enforces by
            # check here. In a real harness, routing goes through a sandbox.
            if entry.get("replay_mode") != "stub_only":
                raise SystemExit(f"tool {s['tool_name']} is not in stub_only mode")
            if "return_sequence" in entry:
                idx = retry_cursors[key]
                if idx >= len(entry["return_sequence"]):
                    raise SystemExit(f"return_sequence exhausted for {key}")
                s["return"] = entry["return_sequence"][idx]
                retry_cursors[key] = idx + 1
            else:
                s["return"] = entry["return"]
            replayed_steps.append(s)
        else:
            replayed_steps.append(s)

    return {
        "trajectory_id":   original["trajectory_id"] + "-replayed",
        "task_family":     original["task_family"],
        "replay_pins":     pins,
        "steps":           replayed_steps,
    }


if __name__ == "__main__":
    out = replay(Path(sys.argv[1]), Path(sys.argv[2]))
    print(json.dumps(out, indent=2))
```

Run it:

```bash
python3 replay.py pins/replay_context.json fixtures/original_trajectory.json > fixtures/replayed_trajectory.json
```

**Expected output:** `fixtures/replayed_trajectory.json` written; stderr is empty; the pin-check passed; every tool call found a recorded return; the three retries on `crm.lookup` consumed the three-entry `return_sequence` in order.

### Step 7 — Verify the replay produced the same scorer verdict

Score both trajectories and compare the `flagged_verdicts`:

```bash
python3 - <<'PY'
import json
from pathlib import Path
from scorer import score

ref = json.loads(Path("fixtures/reference.json").read_text())
orig_verdict = score(json.loads(Path("fixtures/original_trajectory.json").read_text()), ref)
repl_verdict = score(json.loads(Path("fixtures/replayed_trajectory.json").read_text()), ref)
print("original:", orig_verdict["summary"]["flagged_verdicts"])
print("replayed:", repl_verdict["summary"]["flagged_verdicts"])
print("match:   ", orig_verdict["summary"] == repl_verdict["summary"])
PY
```

**Expected output:**

```
original: ['missing_required_tool', 'unauthorised_tool']
replayed: ['missing_required_tool', 'unauthorised_tool']
match:    True
```

If the two `summary` blocks are not equal, one of the six pins drifted. Common suspects:

- A tool-return entry was edited after Step 5 — the model stub now sees a different observation.
- The recorded argument was not canonicalised — `tool_by_key` lookup missed.
- A retry cursor was not advanced — the next crm.lookup saw a stale return.

Chapter 07's framing is: if the replay disagrees with the original, **backtrack to the pin that caused the drift**. Do not smuggle a "well, that's close enough" through a review.

### Step 8 — Bisect by one pin at a time — the system prompt

Chapter 07 Step 4 (Bisect): vary one pin at a time. First bisection: swap the ambiguous planner prompt for a tightened one. Nothing else moves.

```bash
python3 - <<'PY'
import copy, json
from pathlib import Path

orig = json.loads(Path("fixtures/original_trajectory.json").read_text())
tightened = copy.deepcopy(orig)
tightened["trajectory_id"] = "traj-replay-source-prompt-fix"
# Rewrite the planner's system prompt everywhere it appears.
new_system = ("support planner v4: sql_query is for ad-hoc internal "
              "analytics only; use docs.search for user-facing retrieval.")
for s in tightened["steps"]:
    if s["kind"] != "LLM":
        continue
    for m in s["messages_in"]:
        if m["role"] == "system":
            m["content"] = new_system
# A real bisection ALSO rewrites the planner's subsequent tool choices;
# in this lab the stubbed model output changes to reflect the new prompt.
tightened["steps"][0]["messages_out"] = [{
    "role": "assistant",
    "tool_calls": [{"name": "docs.search",
                    "args": {"query": "order 12345 status", "top_k": 3}}],
}]
# And the step-2 tool call changes shape accordingly.
tightened["steps"][1] = {
    "step_id": 2, "span_id": "tool.docs_search", "kind": "TOOL",
    "tool_name": "docs.search",
    "arguments": {"query": "order 12345 status", "top_k": 3},
}
# The planner also now decides to open_ticket after the crm.lookup chain
# resolves — that is the required downstream tool in the DAG.
open_ticket_step = {
    "step_id": 100, "span_id": "tool.open_ticket", "kind": "TOOL",
    "tool_name": "open_ticket",
    "arguments": {"order_id": 12345, "reason": "no_shipping_update"},
}
# Insert right before the final llm.reply.
tightened["steps"].insert(-1, open_ticket_step)
# Add a corresponding LLM(plan.4) step before open_ticket so the trajectory
# still alternates LLM/TOOL. Keep it minimal — captured model calls will
# supply its output.
tightened["steps"].insert(-2, {
    "step_id": 101, "span_id": "llm.plan.4", "kind": "LLM",
    "model": "anthropic/claude-opus-4-7",
    "messages_in":  [{"role": "system", "content": new_system},
                     {"role": "user",   "content": "Where is my order 12345?"}],
    "messages_out": [{"role": "assistant",
                      "tool_calls": [{"name": "open_ticket",
                                      "args": {"order_id": 12345,
                                               "reason": "no_shipping_update"}}]}],
})
Path("fixtures/tightened_trajectory.json").write_text(json.dumps(tightened, indent=2))
PY
```

Now you need to update the recorded tool-response table to cover the *new* step 2 (docs.search instead of sql_query). This is chapter 07's point about replay harness completeness: a new tool call needs a new recorded return, or the replay fails fast.

```bash
python3 - <<'PY'
import hashlib, json
from pathlib import Path

# Append the new docs.search and open_ticket tool-response entries.
entries = json.loads(Path("recorded/tool_responses.json").read_text())
entries.append({
    "step": 2, "tool": "docs.search",
    "args": {"query": "order 12345 status", "top_k": 3},
    "return": [{"id": "doc-91", "score": 0.78,
                "content": "policy: delayed-shipment escalation"}],
    "latency_ms": 400, "status": "ok", "replay_mode": "stub_only",
})
entries.append({
    "step": 100, "tool": "open_ticket",
    "args": {"order_id": 12345, "reason": "no_shipping_update"},
    "return": {"ticket_id": 1247, "status": "open"},
    "latency_ms": 600, "status": "ok", "replay_mode": "stub_only",
})
Path("recorded/tool_responses.json").write_text(json.dumps(entries, indent=2))

# Rebuild the captured model calls table from the tightened trajectory
# so every LLM step has its stub ready. This is simpler and less
# error-prone than editing entries by step_id.
tightened = json.loads(Path("fixtures/tightened_trajectory.json").read_text())
calls = []
for s in tightened["steps"]:
    if s["kind"] != "LLM":
        continue
    calls.append({"step_id": s["step_id"], "model": s["model"],
                  "messages_in": s["messages_in"], "output": s["messages_out"]})
Path("recorded/captured_model_calls.json").write_text(json.dumps(calls, indent=2))

# The messages_hash pin drifts too — recompute it over the tightened
# trajectory's LLM messages. Chapter 07 Step 2 ("Pin") is explicit that
# each experiment gets its own pin set.
buf = b""
for s in tightened["steps"]:
    if s["kind"] == "LLM":
        buf += json.dumps(s["messages_in"], sort_keys=True).encode()
pins = json.loads(Path("pins/replay_context.json").read_text())
pins["messages_hash"] = "sha256:" + hashlib.sha256(buf).hexdigest()
Path("pins/replay_context_prompt_fix.json").write_text(json.dumps(pins, indent=2))
PY
```

Run the replay against the tightened trajectory + new pins:

```bash
python3 replay.py pins/replay_context_prompt_fix.json fixtures/tightened_trajectory.json > fixtures/replayed_prompt_fix.json
python3 - <<'PY'
import json
from pathlib import Path
from scorer import score
ref = json.loads(Path("fixtures/reference.json").read_text())
v = score(json.loads(Path("fixtures/replayed_prompt_fix.json").read_text()), ref)
print(v["summary"]["flagged_verdicts"])
PY
```

**Expected:** `flagged_verdicts` is now `[]`. The planner picks the authorised tool at step 2 (clears `unauthorised_tool`); the required `docs.search` is present (clears `missing_required_tool`); the DAG edge `docs.search → open_ticket` is respected (there was no `ordering_violation` to begin with — chapter 02's note at Step 3). One carefully-scoped bisect, two verdicts cleared on the plan axis.

The trajectory now passes the chapter-02 axes. Residual verdicts (if any) come from the budget side (the retry storm on `crm.lookup` is still there — we did not touch the retry policy) — chapter 07's point: **one experiment, one axis clears, the other axis is still open.**

### Step 9 — Bisect by one pin at a time — side-effect safety

Chapter 07 says side-effectful tools are **never** executed on replay. Confirm your harness enforces this. Add a new tool `pay_invoice` to the trajectory *without* flagging it `stub_only`, and show the replay runner fails closed:

```bash
python3 - <<'PY'
import copy, json
from pathlib import Path
# Add a pay_invoice step to the tightened trajectory.
t = json.loads(Path("fixtures/tightened_trajectory.json").read_text())
t["trajectory_id"] = "traj-pay-invoice-attempt"
t["steps"].insert(-1, {
    "step_id": 99, "span_id": "tool.pay_invoice", "kind": "TOOL",
    "tool_name": "pay_invoice", "arguments": {"order_id": 12345, "amount": 42.00},
})
Path("fixtures/pay_invoice_trajectory.json").write_text(json.dumps(t, indent=2))
# Record pay_invoice WITHOUT stub_only (deliberately wrong).
entries = json.loads(Path("recorded/tool_responses.json").read_text())
entries.append({
    "step": 99, "tool": "pay_invoice",
    "args": {"amount": 42.00, "order_id": 12345},
    "return": {"status": "paid", "txn_id": "T-1"},
    "latency_ms": 50, "status": "ok"
    # NOTE: no replay_mode key set — the runner must refuse this.
})
Path("recorded/tool_responses.json").write_text(json.dumps(entries, indent=2))
PY
python3 replay.py pins/replay_context_prompt_fix.json fixtures/pay_invoice_trajectory.json \
  || echo "runner refused (expected)"
```

**Expected:** the replay runner exits non-zero and prints `tool pay_invoice is not in stub_only mode`. Chapter 07's rule holds: the replay harness is fail-closed on side-effectful tools that have not been explicitly stubbed.

Clean up the bad entry so subsequent steps still work:

```bash
python3 - <<'PY'
import json
from pathlib import Path
entries = json.loads(Path("recorded/tool_responses.json").read_text())
entries = [e for e in entries if e.get("tool") != "pay_invoice"]
Path("recorded/tool_responses.json").write_text(json.dumps(entries, indent=2))
PY
```

### Step 10 — Bisect by one pin at a time — model substitution

Simulate chapter 07's "model retirement" fallback. Change the `response_model_id` pin to a model identifier the provider no longer serves; the replay harness has to flag this as an advisory replay rather than silently continuing against a substitute.

Edit your `replay.py` (or add a wrapper) so that an unknown model id prints a `WARNING: model_substituted=true` line and continues. In practice:

```bash
python3 - <<'PY'
import json
from pathlib import Path
pins = json.loads(Path("pins/replay_context_prompt_fix.json").read_text())
pins["model"]["response_model_id"] = "anthropic/claude-opus-4-7-RETIRED"
pins["harness"]["advisory_notes"] = "model_substituted: provider retired the exact checkpoint"
Path("pins/replay_context_retired_model.json").write_text(json.dumps(pins, indent=2))
PY
```

Then describe (in a short `pins/advisory_notes.md`) why the replay is now **advisory**: it reproduces the shape of the failure, not the trajectory. Chapter 07 is explicit — prompt-bug fixes are still possible under advisory replay; ambiguous-cause fixes are not.

### Step 11 — Final checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `pins/replay_context.json` carries all six pins named in chapter 07 (model identity, sampling parameters, messages_hash, tool_response_table, environment, harness version).
- [ ] `recorded/tool_responses.json` records the **full retry sequence** for `crm.lookup`, in order, not just the final success.
- [ ] Every side-effectful tool entry is marked `replay_mode: "stub_only"`; the runner fails closed on unmarked entries.
- [ ] The replayed trajectory's `flagged_verdicts` matches the original's exactly (Step 7).
- [ ] After the prompt-fix bisect (Step 8), the three plan-axis verdicts clear; the budget-axis verdict (if you ran lab-02 on the same trajectory) does not.
- [ ] An attempt to replay a side-effectful tool without `stub_only` fails fast with a clear error (Step 9).
- [ ] An advisory replay flags `model_substituted=true` and does not pretend to be exact (Step 10).

### Step 12 — Stretch: compose the replay with a budget-axis bisect

Combine this lab with lab-02: run the `budget.py` scorer on the tightened-prompt replay and on an additional bisect where you drop `max_retries_per_tool` from 2 to 1 on `crm.lookup`. Confirm:

- Tightened prompt alone: the plan-axis verdicts clear; `retry_budget_blown` and `latency_over_budget` persist.
- Tightened prompt **and** retry budget tightened: `retry_budget_blown` clears; `cost_over_budget` likely persists (chapter 07 Example makes exactly this observation — the retry fix shrinks the storm but step 2's cost remains).

This is chapter 07 Step 4's "common sequence" — model, prompt, tool catalog, retry policy, harness — made physical. Each experiment is one deterministic replay with one pin changed.

## Where this hands off

- The six-pin JSON + the recorded tool-response table + the stub-model table *is* the replay context exercise-05 asks you to productionise, but with a real Inspect Solver intercepting tool calls and a real model API being called under the replay's pinned parameters.
- The regression-fixture promotion (chapter 07 Step 6) belongs in mod-110; it is explicitly outside this lab's scope. Keep the tightened-prompt trajectory around — it is the regression fixture mod-106's CI gate will later read.
- The six-step post-mortem workflow (**lift, pin, reproduce, bisect, fix, learn**) has been exercised on steps 2 (**pin**), 3+7 (**reproduce**), and 8+9+10 (**bisect**). The **fix** and **learn** steps are the ones exercise-05 walks end-to-end against a real trace and your organisation's regression dataset.
