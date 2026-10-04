# Deterministic Replay and the Post-Mortem Workflow

## Motivation

Chapter 01 named five questions a trajectory scorer must answer; the first four — right steps, right final answer, inside budget, how much partial credit — have been answered by chapters 02 through 06. This chapter answers the fifth: *if the trajectory failed, can we replay it deterministically so the fix is bisectable?*

The need is concrete. A production trajectory from chapter 01's running example fails with `cost_over_budget` and `wrong_tool`. The on-call needs to know whether:

- the planner's prompt is at fault (the model is picking the wrong tool because the system prompt is ambiguous);
- the tool catalog is at fault (two tools have overlapping descriptions);
- a transient flaky dependency drove a retry storm (chapter 04's `retry_dominant_tool`);
- the model version itself regressed (a provider-side change the team did not opt into);
- the harness retry policy was mis-configured (an SLO change that was never reflected in the harness limits);
- the input itself was out-of-distribution (a user turn the eval plan never contemplated).

Each of these fixes lives in a different place. Bisecting them by eye on a single trace is unreliable; bisecting them by running the agent again fresh is non-deterministic because the model, the tool, and the environment all drift. The deliverable this chapter builds is a **replay contract**: a bounded set of inputs that, when fixed, reproduce the trajectory to the token. Once that holds, the on-call can change one input at a time and watch the trajectory change — the empirical bisection method distributed systems use for production bugs applies, finally, to the agent.

## Core concepts

### What has to be fixed for a replay

A trajectory is deterministic if and only if every source of randomness in the loop is pinned. There are six.

**(1) Model identity.** The exact model served at the moment the trajectory ran. OTel's `gen_ai.response.model` is the attribute to read; it may differ from `gen_ai.request.model` if the provider routed to a different checkpoint. Pin to the response-model value, not the request-model value.

**(2) Model sampling parameters.** Temperature, top_p, top_k, seed, logit_bias, max_tokens, stop sequences. All must be identical. Record them under `gen_ai.request.*` at trace time (mod-102 chapter 02); read them at replay time.

**(3) The exact message list sent to the model at each step.** The system prompt (including its version), the user turns, every tool-return the model saw, and any middleware-injected content. If the system prompt was templated at runtime, record the rendered version, not the template. The trajectory in `TaskState.messages` is lossless-enough; the serialised message list in mod-102's `llm.input_messages.*` is the backup.

**(4) Tool returns.** Every value every tool returned at every step. A `crm.lookup` that returned `{"status": "shipped", "eta": "2026-03-01"}` on the original run must return the same bytes on replay. Tool returns are typically the biggest source of non-determinism — the CRM moved on, the retriever's index was rebuilt, the clock advanced — and the mechanism that pins them is the **recorded tool-response table** (next section).

**(5) Environment.** Any state outside the message list that any tool read. Timestamps if a tool asks for "now", random seeds for any stochastic tool, feature flags, A/B assignments, the user's locale, the agent's own id. The surface's trace schema (mod-102 chapter 04) should record each of these as root-span attributes; the replay harness restores them.

**(6) Harness version.** The Inspect version, the Python version, the model-SDK version. A trajectory produced under Inspect v0.4.x and compared to a replay under v0.5.x is not strictly comparable. Pin via the project lockfile and record the Inspect version in the trace header.

If one of the six is unpinned, the replay is advisory — it reproduces the shape of the failure, not the trajectory. Advisory replays are still useful for one class of fix (prompt bugs); they are not sufficient for the ambiguous cases above.

### The recorded tool-response table

The simplest useful artefact. At trace time, the agent harness records every tool invocation along with its exact return value, keyed by `(step_index, tool_name, arguments)`. At replay time, the harness intercepts tool calls and returns the recorded value instead of executing the real tool.

```json
// recorded tool-response table for trace c31f7a…
[
  {
    "step": 2,
    "tool":  "sql_query",
    "args":  {"query": "SELECT * FROM orders WHERE id=12345"},
    "return": {"id": 12345, "status": "in_transit", "eta_days": 2},
    "latency_ms": 712,
    "status": "ok"
  },
  {
    "step": 7,
    "tool":  "crm.lookup",
    "args":  {"order_id": "12345"},
    "return_sequence": [
      {"status": "error", "code": 502, "latency_ms": 2103},
      {"status": "error", "code": 502, "latency_ms": 2008},
      {"status": "ok",    "payload": {"shipment_id": "SHP-0098"}, "latency_ms": 1912}
    ]
  }
]
```

Three design rules:

- **Record on the write-path, not after the fact.** The recording is a Collector-side middleware in mod-102's Collector topology (chapter 03 of that module) or an Inspect sandbox fixture; it is not reconstructed from the trace after the fact. A trace that embeds the tool payloads is enough for most cases, but Collector-side scrubbing may truncate them.
- **Index on `(step_index, tool_name, canonicalised_arguments)`.** The replay harness looks up the recorded return by this key. Use the chapter-02 argument-canonicalisation rules so a semantically-equivalent call still matches a recorded response.
- **Record the full retry sequence.** A tool that was retried has an ordered `return_sequence`; the replay harness yields each entry in order. Collapsing retries into "the final successful return" loses the retry-storm attribution.

A tool whose state changes between recording and replay (`send_email`, `pay_invoice`) is **never** executed on replay — the recorded return is the only legitimate replay-time behaviour. The sandbox discipline from chapter 05 (route side-effectful tools through a stub in eval) is what makes this safe.

### The replay harness, structurally

The replay harness in Inspect terms is a Solver that drives the agent while a `tool_replay_mode` fixture intercepts tool calls:

```
┌──────────────┐
│   Replay    ├────┐
│   Harness    │   │  reads tool-response table + pinned env
└──────────────┘   │
                   ▼
     ┌─────────────────────────┐
     │  model_call(messages,   │  server pinned, params from trace
     │     params, model)      │
     └───────────┬─────────────┘
                 │                            ┌──────────────────────┐
                 ▼                            │  recorded_tool_table │
     ┌─────────────────────────┐              │  (step, name, args)  │
     │ tool_call intercepted ──┼─────▶ lookup(step,name,args) ──┐    │
     └─────────────────────────┘                              return value
                 │                                              ◀────┘
                 ▼
     ┌─────────────────────────┐
     │   next model step       │
     └─────────────────────────┘
```

Three practical requirements:

- **The model has to be re-callable.** A provider that has retired the exact `gen_ai.response.model` identifier makes an exact replay impossible; the fall-back is an approximate replay against the nearest surviving checkpoint, flagged `model_substituted=true`. The product's model-version retention policy (mod-101 chapter 06 of the plan; mod-112 at program altitude) is what determines how long exact replay is possible.
- **The random seed has to be propagated.** For providers that honour `seed`, pin it. For providers that do not (currently most), expect the model-call to be deterministic in expectation but stochastic in detail; the replay may diverge on specific tokens even when the trajectory is the same.
- **Tool interception must be total.** A tool call that falls through to the real tool during replay silently invalidates the run. Make the interception layer fail-closed — unknown `(step, tool, args)` triples are an error, not a pass-through.

### Snapshotting beyond tool returns

Three sources of state that are not tool returns but matter for replay:

- **Vector index snapshot.** A RETRIEVER tool whose return depends on an index that is updated continuously will not replay unless the index is pinned. In practice: either snapshot the index to a content-addressed store at trace time and record its hash, or treat the retriever's *return value* as the recorded tool-response (chapter 02 recommended the latter via equivalence classes on retrieval result sets).
- **Random-seed propagation.** Agents that generate internal randomness (sampling for a candidate, a mix policy) have to accept a seed. The surface's agent code owns propagation; the replay harness provides the seed.
- **Clock pinning.** Tools that return "now" — `current_time`, `today_date`, `is_business_hours` — produce different outputs on different days. Pin via a `clock_at_record_time` field in the trace header and inject it into every tool that reads a clock. For tools that do not expose a clock-injection point, treat clock-reading as part of the tool-return record.

### The post-mortem workflow

A workflow is a sequence of steps a human-plus-tools follows after a trajectory failure fires. The six steps:

**Step 1 — Lift.** Pull the failing trajectory's trace from mod-102's backend and the eval log from Inspect (chapter 05). The scorer's verdict is already attached; the `flagged_verdicts` list points at the family.

**Step 2 — Pin.** Materialise the replay context: tool-response table, env snapshot, model identity, sampling params, harness version. Chapter 05's `Sample.metadata.replay_pin` is the right place to store the materialised context. If any of the six pins is missing, flag the replay as advisory and continue — some fixes are still possible.

**Step 3 — Reproduce.** Run the Inspect Task in replay mode against this one Sample. Confirm the reproduced trajectory matches the original on `flagged_verdicts` and on the dominant attribution. If the reproduction diverges, the trace is missing a pin; backtrack to Step 2.

**Step 4 — Bisect.** Vary one pin at a time. The common sequence:
- *Model version*: unpinned → try the current primary model. Does the trajectory still fail?
- *System prompt*: swap in the proposed-fix prompt. Does the trajectory recover?
- *Tool catalog*: remove the tool the planner wrongly picked. Does the planner pick a right alternative?
- *Retry policy*: tighten `max_retries_per_tool`. Does the retry storm shrink?
- *Harness version*: upgrade Inspect. Does the limit-handling change?

Each experiment is one deterministic replay with one pin changed; the reproducible diff on `flagged_verdicts` and on budget axes is what rules a hypothesis in or out.

**Step 5 — Fix.** Open the PR that implements the winning hypothesis. The eval-plan file (mod-101) is often the right scope; chapter 02's reference-trajectory DAG often is too. For CI gating, mod-106 adds this trajectory to the regression suite.

**Step 6 — Learn.** Promote the fixture into the mod-110 regression dataset so the exact failure cannot recur silently. If the failure mode is new, add a verdict to chapter 03's taxonomy and backfill the eval plan. If the attribution surfaced a dashboard gap (chapter 04's attribution block was wrong), fix the scorer, not the dashboard.

### Attribution in a post-mortem

The chapter-04 attribution block is where the on-call starts. A verdict of `cost_over_budget` without a `cost_dominant_span` and a `retry_dominant_tool` is a bug report with no stack trace. In the running example, the attribution immediately pointed at `tool.crm.lookup` with two 502s — the first hypothesis is the dependency, not the agent. The deterministic replay confirmed the dependency failure deterministically (the recorded tool-response table had the 502s), which let the on-call fix the retry policy independently of the planner prompt.

A good post-mortem traces the fix back to the attribution:
- "The scorer told us `wrong_tool at step 2`; the replay confirmed `sql_query` was picked under the current planner prompt; the fix is a tool-description edit that disambiguates `sql_query` from `docs.search`; the regression suite now covers this case."

A bad post-mortem jumps from the symptom (`wrong_tool`) to a speculative fix (new prompt) without the deterministic replay — and the symptom returns a week later from a different cause.

### When replay is impossible

Four cases where exact replay cannot be achieved and the right disposition is to say so, not to pretend:

- **Model retirement.** The exact `gen_ai.response.model` is no longer served. Falls back to approximate replay with a flagged `model_substituted`.
- **Side-effect in the loop.** The agent's run included a `pay_invoice` that cannot be replayed and whose "return" was dependent on the real downstream state. The post-mortem is still useful for every other pin; the specific side-effect step is marked `non_replayable`.
- **Tool with external, uncaptured state.** A tool that reads a database that is now in a different state and whose return was not recorded at trace time. The fix is to make the tool-response table a hard requirement at the Collector (mod-102 chapter 06 scrubbing has to leave the tool payload intact for replay-eligible traffic).
- **PII scrubbing destroyed the pin.** The tool payload was redacted at the Collector before the eval-data platform received it. The sampling policy (mod-102 chapter 06) has to carve out a replay-eligible retention tier for failing trajectories or the post-mortem workflow quietly degrades over time.

### Replay vs. shadow / canary / A/B

The three sibling evaluation modes it is useful to distinguish:

- **Replay (this chapter).** One trajectory, deterministically reconstructed. For bisecting a single failure.
- **Shadow run (chapter 04 / mod-107).** New agent runs against production traffic *in parallel* with the live one; serves no user. For comparing distributions before promotion.
- **Canary (chapter 04 / mod-107).** New agent serves a small percentage of real users. For catching tail behaviour the shadow could not surface.
- **A/B (mod-107).** Head-to-head comparison for a decision requiring statistical significance.

Replay is the only one of the four that gives *per-trajectory* determinism; the other three gain their statistical power from sample size. The post-mortem workflow needs replay; the rollout decision usually needs one of the other three.

### How this ties into the rest of the module

Chapters 02 and 03 produced the verdict with `flagged_verdicts` and the per-step records. Chapter 04 produced the attribution block (`cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool`). Chapter 05 produced the Inspect Task that the replay harness drives. Chapter 06 shaped the regression suite that Step 6 of the post-mortem workflow updates. This chapter closes the loop: the failing trajectory becomes a reproducible fixture, the fixture becomes a regression test, and the next regression that would have shipped is blocked by the CI gate in mod-106.

## Example — bisecting the chapter-01 bad trajectory

The failing trajectory from chapter 01:

- `flagged_verdicts = [wrong_tool, redundant_call, ungrounded_but_correct, cost_over_budget, latency_over_budget, steps_over_budget]`.
- `cost_dominant_span = llm.plan.3` (1288 tokens, $0.0018).
- `latency_dominant_span = tool.crm.lookup.retry.1` (2103 ms).
- `retry_dominant_tool = crm.lookup`.

**Step 1 — Lift.** Trace `c31f7a…` from Phoenix; Inspect log from the nightly `inspect eval` run. Scorer verdict JSON ready.

**Step 2 — Pin.** Replay context materialised:
- Model: `anthropic/claude-opus-4-7` (served, response-model).
- Sampling: `temperature=0.2`, `top_p=0.9`, `max_tokens=512`, no seed.
- Tool-response table: four records (sql_query, docs.search, docs.search.2, crm.lookup with 3-entry return_sequence, open_ticket).
- Env: `clock_at_record_time=2026-10-01T14:32:08Z`, feature flag `fast_path=off`.
- Harness: Inspect v0.5.x (illustrative). <!-- needs-research: pin to a real recent Inspect release when running an actual replay. -->

**Step 3 — Reproduce.** Run the Task in replay mode on this one Sample. Confirm: the reproduced trajectory emits the same six `flagged_verdicts` with the same dominant spans. The retry storm reproduces exactly (same two 502s followed by an OK), because the tool-response table has the sequence. The planner still picks `sql_query` at step 2.

**Step 4 — Bisect.** Four experiments:
- Swap system prompt to the proposed v4 (which adds "`sql_query` is for ad-hoc internal analytics only; use `docs.search` for user-facing retrieval"). Replay. Result: planner picks `docs.search` at step 2; `wrong_tool` verdict clears. Keep.
- Tighten `max_retries_per_tool` from 2 to 1 for `crm.lookup`. Replay with the swapped prompt. Result: `retry_dominant_tool=crm.lookup` still, but the retry sequence now terminates at retry 1 with `status=error`; `retry_budget_blown` clears, `cost_over_budget` persists because the step 2 cost is still high. Keep.
- Remove the second `docs.search`. The scorer's `redundant_call` detector only flags; it does not prevent. The fix is to add a de-duplication rule to the agent's planner. Add a short note.
- Upgrade Inspect to the next release. Replay. Result: no change on `flagged_verdicts`; harness-level change is not the fault.

**Step 5 — Fix.** Open PR: planner-prompt v4; `max_retries_per_tool=1` for `crm.lookup`. The `redundant_call` fix is tracked separately.

**Step 6 — Learn.** Promote the Sample into `support_agent/order_status/v3.3`. Backfill chapter 03's verdict taxonomy if `redundant_call` was not already there (it was). Confirm mod-106's CI gate reads v3.3 before the merge lands.

The post-mortem is one person-hour, not one person-day, because every step was deterministic.

## Common pitfalls

- **Partial pins.** Pinning five of six sources and claiming "deterministic replay" produces a replay that disagrees with the original run and the on-call cannot tell which pin is loose. Pin all six, flag advisory if one is missing.
- **Replay on retired models.** A trajectory that cannot be replayed because the model is retired is a model-version-retention policy failure, not a replay failure. Fix the retention policy; see mod-101 chapter 06 and mod-112.
- **Scrubbed tool payloads.** A Collector-side PII scrubber that redacts tool returns breaks replay for exactly the failing trajectories you need to replay. Carve a replay-eligible retention tier out of mod-102 chapter 06's sampling policy.
- **Executing side-effectful tools during replay.** Never. The recorded return is the only legitimate replay behaviour; the sandbox discipline enforces it.
- **One-at-a-time bisect inverted.** Changing two pins at once and claiming "the trajectory recovered" tells you the combination worked, not which pin mattered. One pin per experiment.
- **Not updating the regression suite.** Fixing the trajectory without adding the fixture to mod-110 means the exact failure can recur unchallenged. Step 6 is non-optional.
- **Treating `flagged_verdicts` as the only signal.** Attribution matters too. A `wrong_tool` with no `cost_dominant_span` is incomplete; a `cost_over_budget` with no attribution is unactionable.

## Summary

- Deterministic replay pins six sources: model identity (served, not requested), sampling params, exact message list, tool returns (recorded table with full retry sequences), environment (clock, feature flags, locale), and harness version.
- The **recorded tool-response table** is the central artefact: `(step_index, tool_name, canonicalised_arguments) → return` (or return-sequence for retried tools). Side-effectful tools are never re-executed on replay.
- The replay harness runs as an Inspect Solver that intercepts tool calls; interception is fail-closed on unknown keys.
- Beyond tool returns, pin retrieval indices (via content hash), random seeds (propagated from the harness), and the clock.
- Four cases make exact replay impossible: model retirement, side-effect-in-the-loop, uncaptured external state, PII scrubbing that destroyed the pin. For each, name the fallback explicitly.
- The six-step post-mortem workflow: **lift, pin, reproduce, bisect, fix, learn**. Each experiment changes one pin at a time; the fix lands in the eval plan or the agent code; Step 6 promotes the fixture into the mod-110 regression dataset.
- The attribution block from chapter 04 (`cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool`) is where the on-call starts; a verdict without attribution is unactionable.
- Replay is per-trajectory determinism; shadow / canary / A/B are per-distribution statistics. The post-mortem needs the former; the rollout decision usually needs the latter.
- Pitfalls — partial pins, retired models, scrubbed tool payloads, side-effectful replays, two-at-a-time bisects, skipping Step 6 — are mechanical to avoid once named.

That closes the module: the scorer / budget / replay triad is complete. The exercises walk the triad end to end on the support-agent surface; the labs wire it into Inspect; mod-106 wires the output into CI; mod-107 wires the attribution into the online loop.
