# exercise-01: Trajectory Scorer With Tool Call And Budget

**Estimated effort:** 3 hours

## Objective

Build the two scorers that answer trajectory questions 1 and 3 from chapter 01: *did the agent take the right steps?* and *did it do it inside the allowed budgets?* The deliverable is a Python module that reads one captured trajectory (as JSON produced by mod-102's trace exporter) and emits the per-step verdict shape from chapter 02 plus the per-trajectory budget verdict from chapter 04. The scorer stays harness-agnostic — exercise-03 is where you wrap it inside Inspect.

By the end you will have the raw material the rest of the module's scorer stack composes on. Exercise-02 reads your output and rolls it into the partial-credit rubric; exercise-03 reruns it inside Inspect; exercise-05 feeds it a replayed trajectory and asks it to reproduce the same verdict.

## Prerequisites

- Chapters 02 and 04 of this module.
- Python 3.11+. Standard library plus `pydantic` (optional) is enough; no model API calls are required for this exercise.
- One captured agent trajectory in a JSON-serialisable form. Options, in increasing order of effort:
  - **Fixture provided with the module.** Use `exercise-01/fixtures/support_agent_bad_trajectory.json` (chapter 01's running example, canned). If your checkout does not include it, synthesise one by hand to the schema below.
  - **Your mod-102 trace export.** Export a span tree from your Phoenix / Langfuse / Weave instance from the `AGENT` root downward and flatten to the schema below. Keep the AGENT, LLM, TOOL, RETRIEVER, and GUARDRAIL spans.
  - **A trajectory from a tiny agent you write.** 20 lines of Python plus two stub tools, deterministic enough to inspect by eye.

## Choose the task family

Use the running support-agent surface from chapter 01 unless you have a workplace surface you can bring (with your employer's authorisation; redact anything sensitive before you commit). The task family is `order_status_not_shipped_policy`.

The reference trajectory (DAG form) for this task family — repeated here for convenience; the real one lives in mod-110:

```yaml
# support_agent/order_status_not_shipped_policy/v3.2
form: DAG
nodes: [docs.search, crm.lookup, open_ticket]
edges:
  - [docs.search, open_ticket]
  - [crm.lookup,  open_ticket]
required_set: [docs.search, open_ticket]
allowed_set:  [docs.search, crm.lookup, open_ticket]
argument_rules:
  docs.search.query:
    level: 3-semantic
    rule: token_set_includes [order, status]
  open_ticket.reason:
    level: 2-exact
    rule: in {no_shipping_update, shipping_delay, delivery_exception}
budgets:
  cost_usd_ceiling:      0.02
  time_limit_s:          8
  max_steps:             5
  max_tool_calls:        4
  max_retries_per_tool:  1
price_table_version: v12
```

## Trajectory schema (what your scorer reads)

```json
{
  "trajectory_id":      "c31f7a…",
  "task_family":        "order_status_not_shipped_policy",
  "reference_version":  "v3.2",
  "agent": {
    "span_id":   "s_root",
    "start_ns":  1725192728000000000,
    "end_ns":    1725192742200000000,
    "app.cost_usd": 0.219
  },
  "steps": [
    {
      "step_id":  1,
      "span_id":  "s_llm_plan",
      "kind":     "LLM",
      "app.step": "plan",
      "llm.response.model": "anthropic/claude-opus-4-7",
      "tokens": {"input": 652, "output": 224},
      "cost_usd": 0.0009,
      "latency_ms": 500,
      "tool_calls": [
        {"id": "tc1", "name": "sql_query",
         "arguments": {"query": "SELECT * FROM orders WHERE id=12345"}}
      ]
    },
    { "...": "one record per LLM, TOOL, RETRIEVER span" }
  ]
}
```

If your captured trace uses OpenInference indexed attributes directly, write a small adapter — do not teach the scorer two shapes.

## Requirements

Produce a directory `mod-103/exercise-01/` in your working repo containing:

1. **`scorer/per_step.py`** — a function `score_per_step(trajectory, reference) -> list[StepVerdict]` implementing chapter 02:
   - Name matching with alias table, MCP namespace handling, retry collapsing, NFKC + casefold normalisation.
   - Argument matching at Levels 1, 2, and 3. Level 4 and 5 are **not required**; stub them with `NotImplementedError` and a comment pointing at mod-104.
   - Ordering with `graphlib.TopologicalSorter`. On failure, name the violated edge.
   - Return per-step verdicts in the exact JSON shape chapter 02 specifies, including `step_id`, `span_id`, `tool_name`, `arguments`, `order`, `cost_ok` (leave `true` by default; your budget scorer may rewrite it), and `verdict`.
2. **`scorer/budget.py`** — a function `score_budget(trajectory, reference) -> BudgetVerdict` implementing chapter 04:
   - All four families (cost, latency, step count, retry rate).
   - Wall-clock latency from the AGENT span; carry critical-path and sum-of-step as diagnostics.
   - Charge the right model: use `llm.response.model`, not request.model. Separate input / output tokens.
   - Linear decay to zero at 2× over any ceiling; `budget_credit = min across families`.
   - Attribution block: `cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool`.
   - Emit the chapter-04 verdict JSON including `flagged_verdicts`.
3. **`scorer/cli.py`** — a one-screen CLI: `python -m scorer.cli --trajectory <path> --reference <path> --price-table <path>`. Prints the per-step list and the budget verdict as pretty JSON.
4. **`tests/test_per_step.py`** and **`tests/test_budget.py`** — table-driven tests with at least:
   - a trajectory where everything passes (positive control);
   - the chapter-01 bad trajectory (the running example — expect `wrong_tool`, `redundant_call`, four budget verdicts);
   - a trajectory with a collapsed retry (verify it does not count twice);
   - a trajectory with parallel tool calls (verify it counts as one step and N tool calls);
   - a trajectory with a semantic-equivalent `docs.search` query (verify Level 3 accepts it where Level 2 rejects).
5. **`README.md`** — 200–300 words describing the scorer's inputs, outputs, and intentional non-goals (judge grading, Inspect integration, replay).

## Starter guidance

- Start with the trajectory schema. Everything else is downstream.
- The name matcher is small; write it first and lean on it from the argument matcher ("if the name did not match, do not short-circuit the argument scorer — chapter 02 explicitly requires full attribution").
- Keep the Level-3 argument matcher narrow: one semantic rule per field, not a general-purpose equivalence engine. The rule table from the reference-trajectory YAML tells you exactly which fields need which level.
- For the DAG matcher, build the `TopologicalSorter` from the reference edges and consume observed step names into it; on a `CycleError` or on an observed name that violates the partial order, emit the specific violated edge.
- For the budget scorer, the price table is `{model_id: {input_per_1k_tokens, output_per_1k_tokens, cached_input_per_1k_tokens}}`. Pass it in; do not hard-code.
- `app.cost_usd` on the AGENT span is the ground truth sum; your per-step sum should equal it within rounding error. Add an assertion.
- Do **not** emit a scalar score. The rubric in exercise-02 does that; this scorer's output is a structured verdict.
- Capture the price-table version and the reference version in your output. The replay exercise reads them.

## Acceptance criteria

You are done when:

- The CLI runs end-to-end on the provided bad-trajectory fixture and emits the expected `flagged_verdicts`: `wrong_tool`, `redundant_call`, `cost_over_budget`, `latency_over_budget`, `steps_over_budget`, `retry_budget_blown` (the exact list depends on the fixture; justify any deviation in the README).
- All five table-driven tests pass.
- `score_per_step` emits one record per step (not per span), with sub-scores for name, arguments, order, and `cost_ok`.
- `score_budget` emits per-family credits, `flagged_verdicts`, and the attribution block. All three attribution fields are non-null on the bad-trajectory fixture.
- `budget_credit` is pinned to 0 for the bad-trajectory fixture (one or more families is 2× over) and is 1.0 for the happy-path positive control.
- No part of the scorer calls a model. No part of the scorer mutates its inputs.
- Running the scorer twice on the same inputs produces byte-identical output (deterministic, no wall-clock).
- The README names the three things this scorer deliberately does *not* do (judge grading, Inspect wrapping, replay).

## Stretch goals

- Add a **programmatic** reference form alongside the DAG. Author one task-family case where the correct trajectory branches on the retrieval score (e.g., `open_ticket` is required iff no retrieved doc scored ≥ 0.7). Verify your scorer reads a Python predicate instead of a static DAG.
- Add a **Level 4 AST** argument matcher for a hypothetical `sql_query` tool using Python's `ast` module (parse → compare canonicalised). Note where it would fail-closed if the SQL did not parse.
- Add an **equivalence-class** registry for non-deterministic observations (retrieval result order, timestamps). Verify two trajectories that differ only in document order score identically.
- Add a `--price-table-version` CLI flag and a sanity check that the trajectory's recorded version matches it; refuse to score if the versions diverge.

## What this exercise does *not* cover

You are not writing the rubric (exercise-02), you are not wrapping any of this in Inspect (exercise-03), you are not auditing public benchmarks (exercise-04), and you are not replaying anything (exercise-05). Stay narrow: one trajectory in, two structured verdicts out, no scalar.
