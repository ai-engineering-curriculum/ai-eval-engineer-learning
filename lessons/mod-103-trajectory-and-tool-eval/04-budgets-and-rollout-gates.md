# Trajectory Budgets and Rollout Gates

## Motivation

Chapters 02 and 03 answered "did the agent take the right steps and land on the right answer?" This chapter answers the third of chapter 01's five questions — *did the agent do it inside the allowed budgets?* — and converts the answer into a rollout gate that mod-106 (CI) and mod-107 (online eval) consume.

Budgets are not optional on an agentic surface. The retry storm from chapter 01's bad trajectory produced a correct reply at 10× the SLO cost. A rubric that scored only on correctness would ship that trajectory. A budget scorer catches it; a rollout gate stops the canary that is producing it in volume. Three failure modes make this concrete:

- **Cost blow-up.** A planner that silently prefers a Level-5 judge over a Level-2 exact-match (chapter 02) can multiply trajectory cost by an order of magnitude without changing a single reply. Only the roll-up of `app.cost_usd` catches it, and only before it hits a canary at scale.
- **Latency blow-up.** A tool whose p50 is fine but whose p99 is 20× the SLO (a flaky CRM, a cold-start retriever) stays invisible in a mean-score dashboard. The budget scorer records latency per step and names the span that blew the budget.
- **Step-count blow-up.** An agent that is one prompt refactor away from an infinite tool-use loop will produce correct replies until it does not. `max_steps` and `max_tool_calls` are first-class harness controls (<https://inspect.aisi.org.uk/errors_and_limits.html>) and the budget scorer records both the limit and the observed count on every trajectory.

The chapter builds the budget scorer end to end: what to measure, how to aggregate, how to turn the aggregates into rollout gates, and how the gates compose with mod-106's CI check and mod-107's online loop.

## Core concepts

### The four budget families

Every budget this chapter talks about belongs to one of four families. Each family has a per-step record, a per-trajectory roll-up, and one or more SLO ceilings.

**Cost.** Dollars (or token-equivalent cost) per trajectory. Per-step: `app.cost_usd` on each LLM span (prompt + completion tokens × model price) and each TOOL span (per-call price for priced tools; zero for free in-house tools). Per-trajectory: sum across all children of the AGENT span. SLO ceiling: `cost_usd_ceiling` per task family — a number mod-111 owns the derivation of, and the budget scorer treats as given.

**Latency.** Wall-clock seconds per step and per trajectory. Per-step: `span.end_time - span.start_time` on each LLM, RETRIEVER, TOOL, and GUARDRAIL span. Per-trajectory: wall-clock between the AGENT span's start and end, which is *not* the sum of child durations because parallel tool calls overlap. Both are useful: the per-step view attributes a slow step; the trajectory-level view is what the user actually experiences. SLO ceiling: `time_limit` per trajectory, plus per-step `step_time_limit` for critical tools.

**Step count.** Number of agent-decision steps in the trajectory, where a step is one iteration of the agent loop (chapter 01). Per-step: implicit — one record per `(LLM, TOOL)` pair plus the terminal LLM. Per-trajectory: integer count. SLO ceilings: `max_steps`, `max_tool_calls`, `max_messages` — Inspect's harness supports `message_limit`, `token_limit`, `time_limit`, `working_limit` directly (<https://inspect.aisi.org.uk/errors_and_limits.html>).

**Retry rate.** Number of harness retries per tool, per trajectory. Per-step: a count of consecutive `(tool_name, arguments)` calls that differ only in success status (chapter 02 collapsed these for name matching; the budget scorer counts them). Per-trajectory: `max_retries_observed = max over tools (retry_count)` and `total_retries`. SLO ceilings: `max_retries_per_tool` and `max_total_retries`.

The budget scorer emits all four on every trajectory. The scorer never *infers* a budget; the ceilings are read from the task family's eval plan (mod-101) and are versioned alongside the reference fixtures (mod-110).

### What `app.cost_usd` has to carry

The cost column is where most of the subtlety lives. Three precision rules:

- **Charge the right model.** `gen_ai.response.model` (not `gen_ai.request.model`) is the model that was actually billed. OTel's semantic conventions distinguish the two for exactly this case (<https://opentelemetry.io/docs/specs/semconv/gen-ai/>); mod-102 chapter 02 covers the mirror attribute under OpenInference. The budget scorer reads `response.model`.
- **Separate input / output tokens.** Current model pricing is asymmetric — output tokens are typically 3–5× the input price. The scorer multiplies `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` by the per-model input / output rates from a versioned price table. Do not store a flat `cost_per_token`; it is wrong for every modern model.
- **Record cached-input tokens separately.** Providers that bill cached prompt-prefix reads at a reduced rate (Anthropic prompt caching, OpenAI prompt caching) expose a cached-input count. Carry it in the price table and in `app.cost_usd` arithmetic; a cost model that ignores cache credits a surface that aggressively caches as "cheap" when it is actually average, and vice versa when the cache misses in production.

The price table lives in the eval-data platform (mod-110), is versioned, and is a public artefact of the surface. A trajectory scored against price-table-v12 and compared to a canary scored against price-table-v13 is comparing apples and oranges; the version is part of the verdict.

### Latency arithmetic for parallel tool calls

A step with parallel tool calls (OpenAI's `tool_calls` array or Anthropic's multiple `tool_use` blocks in one assistant turn) produces multiple TOOL spans whose durations overlap. Three latency numbers from the same trajectory:

- **Wall-clock trajectory latency.** `agent.end_time - agent.start_time`. The number the user sees. The one the SLO is set against.
- **Critical-path latency.** The longest span-dependency chain through the DAG, computed from span start/end times and parent/child links. For sequential agents this equals wall-clock; for parallel agents it is strictly less. Useful for "what is the best this agent could do if every parallel branch finished at once?".
- **Sum-of-step latency.** `sum of child span durations`. For parallel branches this *overcounts*; it is close to the number a cost model wants to see if time is being billed per-step (serverless agent runtimes), and misleading otherwise. Carry it but do not SLO on it.

The scorer records all three on every trajectory and SLOs on wall-clock. The others are diagnostic; the on-call reads them when wall-clock drifts and the per-step view does not immediately say why.

### Step count, tool count, retry count

These three are integers; the pitfalls are definitional, not arithmetic.

- **Step** is one iteration of the agent loop (one action-selection LLM call plus zero or more tool executions). A step with no tool call (the terminal reply) is counted. A step with parallel tool calls is **one** step, not N.
- **Tool call** is one entry in a step's `tool_calls` array. A step with parallel calls contributes N to the tool-call count. A collapsed retry (chapter 02) is **one** tool call plus a retry count, not multiple tool calls — the retry is a harness event, not a plan decision.
- **Retry** is one re-execution of the same `(tool_name, arguments)` tuple inside the same step, triggered by the harness's retry policy. `max_retries_per_tool` applies here. User-visible re-asks (the agent decided to re-call a tool with slightly different arguments) are not retries; they are additional tool calls.

The definitions above are the ones most harnesses — Inspect, LangGraph's `ToolNode` with retry policy, OpenAI's assistants runtime — converge on. Do not redefine them for your scorer; the delta between what your scorer thinks is a "retry" and what the harness thinks is one will produce ghost retries in the budget dashboard.

### The composite budget score

Chapter 03's rubric reads `budget_credit ∈ [0, 1]` as one of the six weighted axes. The budget scorer computes it as follows:

```
budget_credit = min over families of family_credit, where
  family_credit = clamp(1 - over_fraction, 0, 1)
  over_fraction = max(0, observed / ceiling - 1)
```

Example: `cost_usd = 0.219`, `ceiling = 0.020`. `over_fraction = 0.219/0.020 - 1 = 9.95`. `cost_credit = 0`. Any family that is 2× over the ceiling pins the family credit at 0 and thus `budget_credit = 0`. Any family at or under its ceiling contributes 1.0; a family at 110% of the ceiling contributes 0.9.

Three design rules:

- **Linear decay, bounded at zero.** A quadratic or soft penalty is tempting ("close is still partial credit"); it hides the right-of-the-SLO tail. Linear decay to zero makes "n× over" legible.
- **Worst-family wins.** Taking `min` across families, not a weighted sum, is deliberate: a trajectory that is 2× over cost and nominal on latency is still a failure, and the trajectory-level credit has to show it. The rubric (chapter 03) applies its own weight across the six axes; this family of axes is already compressed inside `budget_credit`.
- **Pre-register the ceilings.** The ceilings come from the eval plan, not from this scorer. Do not fit a ceiling to the current sample.

The scorer also emits the `flagged_verdicts` for chapter 03 — one per family breach — so downstream aggregation can bucket:

```json
{
  "budget_credit": 0.0,
  "budget_axes": {
    "cost":     {"observed": 0.219, "ceiling": 0.020, "credit": 0.0, "over_fraction": 9.95},
    "latency":  {"observed_wallclock_s": 14.2, "ceiling_s": 8.0, "credit": 0.225, "over_fraction": 0.775},
    "steps":    {"observed": 8, "ceiling": 5, "credit": 0.4, "over_fraction": 0.6},
    "tool_calls": {"observed": 6, "ceiling": 4, "credit": 0.5, "over_fraction": 0.5},
    "retries":  {"observed_max": 2, "ceiling_max": 1, "credit": 0.0, "over_fraction": 1.0}
  },
  "flagged_verdicts": [
    "cost_over_budget",
    "latency_over_budget",
    "steps_over_budget",
    "retry_budget_blown"
  ],
  "attribution": {
    "cost_dominant_span":    "span_id for the LLM span that cost the most",
    "latency_dominant_span": "span_id for the slowest TOOL span",
    "retry_dominant_tool":   "crm.lookup"
  }
}
```

### Rollout gates

The scorer's verdict feeds two rollout gates, one offline and one online.

**Offline (CI, mod-106).** The merge gate reads the eval suite's roll-up:

- `pass_rate >= pass_rate_floor` (default 0.95 for regression suites; see mod-106 for the full set).
- `cost_p95 <= cost_ceiling * (1 + tolerance)` across the suite (default tolerance 0.1).
- `latency_p95 <= time_limit * (1 + tolerance)` across the suite.
- No new `flagged_verdict` in `{wrong_tool, unauthorised_tool, fabricated_claim}` compared to the baseline.

The CI check fails fast on floor-tripped verdicts (chapter 03) and on p95 overshoots. It also writes the per-trajectory records back as `EVALUATOR` spans on the original traces (mod-102 schema) so mod-107's online loop can read the same shape.

**Online (canary, mod-107).** The canary gate reads a sliding window of production trajectories:

- `pass_rate` over the last N minutes against a baseline p-value (two-sample proportion test; mod-107 owns the depth).
- `cost_p95` / `latency_p95` over the window.
- `rate_of_flagged_verdict` for the floor verdicts — if `wrong_tool` rate rises above the baseline band, roll back.
- `retry_rate` as a leading indicator of a flaky dependency; often moves before pass-rate drops.

Canary promotion requires all four signals to clear for a sustained window; a single-point breach triggers a hold, a sustained breach triggers a rollback. The budget scorer is the component that produces the per-trajectory inputs to these; mod-106 and mod-107 are the gates that consume them.

**Shadow runs.** A new model or prompt is run in shadow (serves no user; the scorer records trajectories) before any canary promotion. The budget scorer's verdict for the shadow is compared to the current primary's budget on the same traffic. A shadow that passes correctness but blows cost or latency does not get promoted; the budget axis gates the promotion separately from the quality axis. See mod-107 for the shadow / canary / full-rollout contract.

### Composing with Inspect's harness limits

Inspect's `TaskState` carries `message_limit`, `token_limit`, `time_limit`, `working_limit` (<https://inspect.aisi.org.uk/errors_and_limits.html>). Two differences between harness limits and SLO ceilings matter for this chapter:

- **Harness limits are hard stops.** When `message_limit` is reached, the harness terminates the trajectory with a `limit_exceeded` status; the agent cannot continue. The budget scorer reads this as `terminated_by_limit=message_limit` and attributes the failure to the budget family. SLO ceilings are *soft* in the sense that the trajectory completes but is marked `cost_over_budget` for the rollout gate to consider. Set harness limits comfortably above SLO ceilings so a single violation does not force a termination that confounds the scoring signal.
- **Harness limits catch runaway loops; SLO ceilings catch drift.** The two concerns do not substitute. Use both; have the scorer record which one(s) were hit.

### Attribution and the "find the span" rule

A rollout gate that says "cost over budget" is not actionable. A rollout gate that says "cost over budget; the `llm.plan.3` span cost $0.0014, which is 4× its baseline, dominated by a 1104-token prompt; see trace c31f7a…" routes the fix.

The attribution block in the verdict (`cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool`) is what mod-107's dashboards group by; the resulting Pareto chart ("5% of trajectories are driving 60% of this week's cost regression; all 5% are the `llm.plan.3` step of the `order_status` family") is the artefact an on-call opens on page one.

The attribution rule is simple: for each budget family, the dominant span is the single span whose family-metric is the largest contributor to the trajectory total. For cost, that is the span with the largest `app.cost_usd`. For latency, the slowest span on the critical path. For retries, the tool with the highest retry count. Record the span IDs, not just the names — mod-107's trace viewer deep-links on them.

### Budget-family rollups across many trajectories

Chapter 03 cautioned against mean scores. The same rule applies to budget signals, with one addition: use percentiles. A mean cost over a sample hides a long tail where 1% of trajectories drive 50% of the spend. Four roll-ups to carry on every rollout dashboard:

- `cost_p50`, `cost_p95`, `cost_p99`.
- `latency_p50`, `latency_p95`, `latency_p99` (wall-clock).
- `steps_p95`, `tool_calls_p95`, `max_retries_p99`.
- `rate_of_flagged_verdict` for each budget verdict.

p95 and p99 are the ones that most often move first; mean follows weeks later.

## Example — gating a canary on the support agent

Suppose the support-agent surface runs a canary promotion for a prompt refactor that is intended to shrink `llm.plan` turns. The budget-side signals the canary gate evaluates:

- Shadow-run roll-up (1,000 trajectories, prior week's traffic):
  - `pass_rate=0.962` vs baseline 0.955 (acceptable).
  - `cost_p95=$0.021` vs baseline $0.024 (improvement).
  - `latency_p95=5.8s` vs baseline 5.2s (regression within tolerance).
  - `rate_of_flagged_verdict{cost_over_budget}=0.014` vs baseline 0.021 (improvement).
  - `rate_of_flagged_verdict{wrong_tool}=0.003` vs baseline 0.003 (unchanged).
  - Attribution: `cost_dominant_span` shifted from `llm.plan.3` to `llm.reply` — expected given the refactor.

Verdict: shadow passes, promote to 5% canary. On canary, the gate reads a 15-minute sliding window:

- Minute 5: `cost_p95=$0.023` — within band.
- Minute 10: `cost_p95=$0.040`, `rate_of_flagged_verdict{cost_over_budget}=0.057` (prior-window baseline 0.021). Single-breach — hold the canary at 5%, do not expand.
- Minute 15: breach sustained, same families, `latency_p95` also drifting up. Rollback triggered.

The post-mortem (chapter 07's replay workflow) extracts the 20 highest-cost trajectories from the canary window, replays them deterministically, and finds that the prompt refactor silently stopped obeying the per-task tool-filter — the model is now calling an extra `summarise` step on every trajectory. The fix is a prompt revert plus a scorer update to flag the `extra_tool` verdict for `summarise` in this task family. The rollout gate's job was to stop the regression at 5%; the budget scorer's job was to produce the per-trajectory inputs the gate acted on.

## Common pitfalls

- **No ceilings in the eval plan.** A scorer that computes `cost_usd` but has no ceiling to compare against produces a dashboard, not a gate. Treat missing ceilings as a mod-101 bug; chase them down before building this scorer.
- **Mean cost / mean latency.** The long tail is where the money is. Carry p95 and p99 always; carry mean as an informational secondary.
- **Ceilings fit to the current sample.** A ceiling derived from "last week's p95" and compared to "this week's p95" is a null hypothesis. Ceilings come from the surface's SLO (mod-101), not from the eval sample.
- **Confusing step and tool-call counts.** A step with parallel tool calls is one step and N tool calls. A collapsed retry is one tool call plus a retry count. Pick the definitions above and keep them.
- **Serializing a parallel latency.** Summing child-span durations across parallel branches overstates wall-clock. Report wall-clock from the AGENT span's own start/end for SLO purposes.
- **Harness limits standing in for SLOs.** `message_limit=20` is not an SLO. If the trajectory terminates because the harness capped messages at 20 and the SLO was 10, you still have a budget breach but the scorer sees `terminated_by_limit` and might ignore it. Set harness limits above SLOs; score against the SLO.
- **No attribution.** "Cost over budget" without a span id is a bug report with no stack trace. The attribution block is load-bearing for mod-107's workflow.

## Summary

- Four budget families: cost, latency, step count, retry rate. Each has per-step records, a per-trajectory roll-up, and SLO ceilings pre-registered in the eval plan.
- Charge the right model (served, not requested), separate input / output / cached token prices, and version the price table alongside fixtures.
- Report wall-clock latency against the SLO; carry critical-path and sum-of-step latency as diagnostics.
- Define step, tool call, and retry per the harness vocabulary. A parallel step is one step and N tool calls; a collapsed retry is one tool call plus a retry count.
- `budget_credit = min over families of linear-decayed credit`. Pin to zero at 2× over any ceiling; `min` across families so one blown family cannot be averaged away.
- The scorer emits per-family credits, `flagged_verdicts` for each breach, and an attribution block (`cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool`).
- Rollout gates: offline (mod-106) reads p95 and floor-verdict rates on the regression suite; online (mod-107) reads the same on a sliding window of canary traffic. A single-point breach holds; a sustained breach rolls back.
- Harness limits (Inspect's `message_limit`, `token_limit`, `time_limit`, `working_limit`) are hard stops; SLO ceilings are soft. Set the former above the latter; score against the latter.
- Report p50 / p95 / p99 for every budget family across trajectories, plus `rate_of_flagged_verdict`. Mean alone hides the tail.

Chapter 05 covers the Inspect agent harness — Task, Sample, Solver, Scorer, Tool, limits — that the scorers from this chapter and chapters 02–03 plug into.
