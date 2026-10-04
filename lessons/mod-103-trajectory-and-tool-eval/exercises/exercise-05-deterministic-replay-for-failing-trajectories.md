# exercise-05: Deterministic Replay For Failing Trajectories

**Estimated effort:** 3 hours

## Objective

Take a failing trajectory produced by exercise-03 (or captured from any mod-102-instrumented agent) and make it **exactly replayable**. The deliverable is a `recorded_tool_response_table`, a materialised six-pin replay context, and a replay harness that reads them and reproduces the trajectory with the same `flagged_verdicts` and the same attribution. Then walk the six-step post-mortem (**lift, pin, reproduce, bisect, fix, learn**) and close the loop by promoting the failing Sample into the regression dataset.

By the end you will have the artefact that makes an agent bug bisectable instead of guessable — the piece chapter 07 argues is non-optional at program altitude.

## Prerequisites

- Exercise-03 complete, with at least one Sample that fails the rubric (your `S5` regression fixture is a good candidate).
- Chapter 07 of this module.
- Inspect installed; your trace backend accessible (or saved exports of the AGENT sub-tree).
- Either Python's `pathlib` + `json` or a small snapshot library for the recorded-table step. No new third-party dependency is required.
- A model API key with a very small budget cap (the replay should use the recorded tool table and avoid the real tools, but the model is called).

## Set up

1. Create `mod-103/exercise-05/` with subdirectories `pins/`, `recorded/`, `replay/`, `logs/`.
2. Export the failing trajectory from exercise-03's log (`inspect view` → export, or copy from `./logs/`).
3. Confirm the trajectory you selected has `pass=false` and a `floor_tripped` verdict. If every exercise-03 run passed, deliberately break the Sample's planner prompt (chapter 07 §Example walks this).

## Author the six pins

Materialise the six pins from chapter 07 as a single `pins/replay_context.json`:

```json
{
  "trajectory_id":   "...",
  "model": {
    "response_model_id": "anthropic/claude-opus-4-7",
    "request_model_id":  "anthropic/claude-opus-4-7"
  },
  "sampling": {
    "temperature": 0.2,
    "top_p":       0.9,
    "max_tokens":  512,
    "seed":        null
  },
  "messages_hash":   "sha256:…",   // hash of the serialised message list for sanity check
  "tool_response_table": "recorded/tool_responses.json",
  "environment": {
    "clock_at_record_time": "2026-10-01T14:32:08Z",
    "feature_flags":        {"fast_path": "off"},
    "locale":               "en-US"
  },
  "harness": {
    "inspect_version": "x.y.z",
    "python_version":  "3.11.9"
  }
}
```

If any pin is missing from the trace, flag it as advisory in a `pins/advisory_notes.md` file; chapter 07's "when replay is impossible" case list is the vocabulary.

## Build the recorded tool-response table

For every tool invocation in the trajectory, record a canonicalised entry:

```json
[
  {
    "step":   2,
    "tool":   "sql_query",
    "args":   {"query": "SELECT * FROM orders WHERE id=12345"},
    "return": {"id": 12345, "status": "in_transit", "eta_days": 2},
    "latency_ms": 712,
    "status": "ok"
  },
  {
    "step":   7,
    "tool":   "crm.lookup",
    "args":   {"order_id": "12345"},
    "return_sequence": [
      {"status": "error", "code": 502, "latency_ms": 2103},
      {"status": "error", "code": 502, "latency_ms": 2008},
      {"status": "ok",    "payload": {"shipment_id": "SHP-0098"}, "latency_ms": 1912}
    ]
  }
]
```

Three rules the chapter made explicit — enforce them in your table-builder:

- **Record the full retry sequence** (not just the final success).
- **Canonicalise arguments** using chapter-02's rules (sort keys, normalise strings, coerce numeric strings).
- **Flag side-effectful tools** (`send_email`, `open_ticket` if your surface actually sends mail / mutates CRM) with `replay_mode: "stub_only"`. Replay will never execute them.

## Author the replay harness

A small Inspect `Solver` (or a plain Python harness if you want to prove the shape first):

- Reads `replay_context.json` and the tool table.
- Pins the model + sampling params via Inspect's `Task(model=..., ...)`.
- Intercepts every tool call: look up `(step_index, tool_name, canonicalised_args)` in the recorded table; return the recorded value. **Fail-closed** on unknown keys.
- Advances the `clock` via a thin wrapper so any clock-reading tool reads `clock_at_record_time`.
- Writes the replayed trajectory next to the recorded one in `logs/`.

Then rerun exercise-02's rubric on the replayed trajectory; confirm the verdict matches the original on `flagged_verdicts`, `floor_tripped`, and the three attribution span IDs.

## Walk the six-step post-mortem

Produce `postmortem.md` documenting all six steps for your chosen trajectory. The structure mirrors chapter 07's example:

1. **Lift.** Which trace, which Inspect log, what the original verdict said. Attach the span IDs that failed.
2. **Pin.** What is in `replay_context.json`. Any advisory pin? If so, which, and what fix would make it full.
3. **Reproduce.** Confirmed exact replay? List the verdict fields that matched and any that drifted (should be none; if any, you have a loose pin).
4. **Bisect.** At least two one-pin-at-a-time experiments. For each: which pin changed, which verdict fields changed, which hypothesis is ruled in / out. Chapter 07's example lists four; two are sufficient for the exercise.
5. **Fix.** The PR you would open. Does it touch the system prompt? The tool catalog? The reference trajectory DAG? The rubric floors? Say which.
6. **Learn.** Promote the Sample into the regression dataset (`support_agent/order_status/vX.Y`); update exercise-03's dataset to include it; verify exercise-02's rubric fails on the regression Sample.

## Requirements

The deliverable directory contains:

1. **`pins/replay_context.json`** — the six-pin materialised context.
2. **`pins/advisory_notes.md`** — any pin that is advisory, with the chapter-07 reason-case list applied.
3. **`recorded/tool_responses.json`** — the recorded tool-response table with retry sequences and `replay_mode` flags.
4. **`replay/harness.py`** — the Inspect Solver (or plain Python) that reads the pins + table and runs the replay.
5. **`replay/verify.py`** — a diff tool that compares two Inspect eval logs (original vs replay) on `flagged_verdicts`, `floor_tripped`, and the three attribution span IDs. Exits non-zero on drift.
6. **`logs/original.json`** and **`logs/replayed.json`** — the two Inspect eval logs. The verify script passes.
7. **`postmortem.md`** — the six-step narrative. Each step is one or two paragraphs; the bisection step shows at least two experiments.
8. **`regression/`** — the fixture promoted into the regression dataset (chapter 07 §Step 6).

## Starter guidance

- Build the recorded tool-response table first. Everything else depends on it.
- The `(step_index, tool_name, canonicalised_args)` key is the hard part. Reuse exercise-01's argument canonicalisation code.
- For `return_sequence`, feed the next entry on each successive call (`crm.lookup` called three times in a row returns entry 0, then entry 1, then entry 2).
- Fail-closed means fail-closed. An unknown key is a test failure, not a silent passthrough; the whole point of the exercise is to catch loose pins.
- The clock pin matters more than it looks. A tool that calls `datetime.utcnow()` without a wrapper will drift on replay. Wrap it; alternatively treat `datetime` reads as part of the tool-return record.
- Keep the bisection one pin at a time. The temptation to swap two things at once will trade exact attribution for "it works now" and defeats the point.
- Record the model substitution case explicitly. If the exact `response.model` is still served, you have an exact replay; otherwise you have an approximate replay with `model_substituted=true`.

## Acceptance criteria

You are done when:

- `python -m replay.harness --context pins/replay_context.json` reproduces the original trajectory and writes `logs/replayed.json`.
- `python -m replay.verify logs/original.json logs/replayed.json` exits 0 — `flagged_verdicts`, `floor_tripped`, and the three attribution span IDs all match.
- The tool table includes a tool that was retried, with a full `return_sequence`, and the replay reproduces the retry pattern.
- The replay harness fails loudly on an unknown `(step, tool, args)` key — demonstrate this with a `tests/test_fail_closed.py`.
- The post-mortem has two bisection experiments, each with a clear verdict-level diff; the fix PR is named and scoped.
- The regression fixture appears in `regression/`, versioned (`vX.Y`), and loading it into exercise-02's rubric produces `pass=false`.
- All six pins from chapter 07 are either present or explicitly marked advisory with a reason.

## Stretch goals

- **Collector-side recording.** Instead of extracting the tool table from a saved trace, write a collector-side OpenTelemetry processor (mod-102 chapter 03) that writes the tool-response table on the fly as a side-channel artefact. Trade-off: throughput vs completeness.
- **Index-snapshot replay.** For a RETRIEVER span, snapshot the vector index to a content-addressed store at trace time and record the hash; replay against the snapshotted index rather than against the recorded return values. Compare the two approaches on one trajectory.
- **Partial-pin recovery.** Deliberately drop one pin (sampling params, say) from `replay_context.json` and show how far you can still get with an advisory replay. Name the specific hypotheses the advisory replay can still rule in or out.
- **Model-substitution report.** Pick a model you can retire from your account's rotation; replay the trajectory with and without the exact model and quantify the drift in `flagged_verdicts`. Argue what retention policy the surface needs (see mod-101 chapter 06 and mod-112).

## What this exercise does *not* cover

You are not building the full observability pipeline (mod-102 chapter 06), you are not wiring the replay harness into CI (mod-106), and you are not sampling production traffic (mod-107). The replay is one trajectory, deterministic, bisectable; the rollout decision uses the per-distribution tools in those other modules.
