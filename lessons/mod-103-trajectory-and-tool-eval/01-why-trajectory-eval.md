# Why Trajectory-Level Evaluation for Production Agents

## Motivation

The moment a surface stops being "one prompt, one completion" and starts being *agentic* — a loop that selects a tool, calls it, reads the return, replans, maybe calls another tool, maybe retries, and eventually emits a final answer — final-answer evaluation stops describing the system. The reply can look correct on the transcript while the *path* the agent took to produce it violated a dependency, retried a broken CRM call four times, cost 10× the budget, or short-circuited a required verification step. Grading only the endpoint hides all of that. The unit that has to be evaluated at application altitude is not the final message; it is the whole **trajectory** — the ordered sequence of steps the agent executed to get there.

This module teaches you to score that trajectory. The raw material has already been produced upstream: mod-101 fixes the SLIs the eval plan promises against (task success, groundedness, safety, cost, latency), and mod-102 produces the *trace* that records what actually happened — the AGENT root, the child LLM spans, the child TOOL spans, their inputs, outputs, token counts, timings. A trajectory scorer is a program that reads that trace and emits a structured verdict on the path. It is the piece the rest of the track then plugs its own machinery into: mod-104's LLM-as-judge is one implementation of a trajectory scorer, mod-106 gates CI on trajectory scorers, mod-107 samples production trajectories and runs scorers online, mod-111 rolls trajectory-level cost and latency into its trade-off dashboard, and mod-112 aggregates trajectory-scorer verdicts into the program-level report. If the trajectory scorer this module produces is wrong or missing, every one of those downstream modules degrades to a proxy that measures the wrong thing.

This first chapter fixes the vocabulary the rest of the module reuses — *trajectory*, *step*, *action*, *observation*, *tool-call*, *tool-return*, *terminal state*, *sub-trajectory*, *budget* — and states the five trajectory questions any scorer this module builds must answer on the trajectory record alone.

## Core concepts

### Vocabulary

The vocabulary is inherited from three primary sources, all of which converged on the same shape: the OpenAI function-calling / tool-use API (<https://platform.openai.com/docs/guides/function-calling>), Anthropic's tool-use API (<https://docs.anthropic.com/en/docs/build-with-claude/tool-use>), the OpenInference span-kind spec (<https://github.com/Arize-ai/openinference/tree/main/spec>), and the UK AI Security Institute's Inspect framework (<https://inspect.aisi.org.uk/>). Reuse these terms verbatim; do not invent your own.

- A **trajectory** is the ordered sequence of steps produced by one run of an agent, from the initial user input to the terminal state. In Inspect this is the `TaskState` record with its `messages` and `output` (<https://inspect.aisi.org.uk/agents.html>). In OpenAI / Anthropic tool-use terms it is the message list that alternates between assistant turns and `tool` / `tool_result` turns until the model returns without a further `tool_calls` / `tool_use` block. In OpenInference the trajectory *is* the sub-tree rooted at the AGENT span.
- A **step** is one iteration of the agent loop: an action-selection LLM call plus, if the model emitted a tool call, the corresponding tool execution and its return. One step in the loop typically produces two spans in the trace (one LLM, one TOOL); a step with no tool call produces one span (the terminal LLM).
- An **action** is what the model decides to do at a step. In the OpenAI schema an action is either a natural-language reply (a `message` with no `tool_calls`) or a `tool_calls` array of `{id, type, function: {name, arguments}}` entries. In the Anthropic schema an action is a `content` block of type `text` or of type `tool_use` with `{id, name, input}`. In Inspect it is a `ChatMessageAssistant` optionally carrying `tool_calls`.
- A **tool-call** is one entry inside an assistant turn's tool-calls list: a `(tool_name, arguments)` pair the runtime is expected to execute. A single action can carry multiple tool-calls (parallel tool use).
- A **tool-return** is the paired result the runtime feeds back to the model on the next turn: OpenAI's `role: "tool"` message with `tool_call_id` and `content`, Anthropic's `role: "user"` message containing a `tool_result` block with matching `tool_use_id`.
- An **observation** is what the agent sees after acting — in practice, the tool-return content, plus any environment change the runtime surfaces (e.g. a `RETRIEVER` result the agent framework injects). The action / observation split is the classical ReAct framing (Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models", <https://arxiv.org/abs/2210.03629>); the OpenAI and Anthropic APIs are one implementation of it.
- A **terminal state** is the last step of the trajectory. It is reached when either (a) the model returns an assistant turn with no tool calls (natural stop), (b) the harness hits a `max_steps` / `max_messages` limit, (c) a budget guard trips (cost, latency, retries — chapter 04), or (d) an explicit `submit` / `end_turn` tool is called. Inspect represents this with `TaskState.completed` plus an `output` (<https://inspect.aisi.org.uk/reference/inspect_ai.solver.html>).
- A **sub-trajectory** is a contiguous slice of steps that constitute a bounded piece of work — the "retrieve then rerank" prefix, the "call CRM then draft reply" suffix, the recursive sub-agent's whole run. Sub-trajectories map to CHAIN or nested AGENT sub-trees in the mod-102 schema and are the granularity at which per-step scorers (chapter 02) attach.
- A **budget** is a bound on trajectory-level resource use — max steps, max tool calls, max wall-clock, max cost in dollars, max retries per tool. Budgets are first-class in Inspect via `message_limit`, `token_limit`, `time_limit`, `working_limit` (<https://inspect.aisi.org.uk/errors_and_limits.html>) and are the object chapter 04 turns into rollout gates.

### How a trajectory maps to a trace

The mod-102 schema (chapter 04) records an agent run as a root `AGENT` span with alternating `LLM` (action selection) and `TOOL` (observation) child spans, terminated by a final `LLM` span that produces the reply. `RETRIEVER` spans appear when the agent's tool is a retrieval; `GUARDRAIL` spans wrap pre- or post-filter checks. `openinference.span.kind` (<https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md>) is the attribute that carries the kind.

Concretely, for the same support-agent surface mod-102 chapter 01 uses — one agent that plans, retrieves, calls `open_ticket` if needed, drafts a reply — a *well-shaped* trajectory looks like this:

```
trace 4b8a5c…
└─ agent.support_agent                            [openinference.span.kind=AGENT]
   ├─ llm.plan                                   [kind=LLM, app.step=plan]
   ├─ retriever.docs.search                      [kind=RETRIEVER, app.step=docs.retrieve]
   ├─ tool.open_ticket                           [kind=TOOL, tool.name=open_ticket]
   ├─ llm.reply                                  [kind=LLM, app.step=reply]
   └─ guardrail.output_policy                    [kind=GUARDRAIL]
```

The trajectory *is* the child list of `agent.support_agent`, ordered by span start time. Each edge is a step transition. The two `LLM` spans are action-selection points (`llm.plan` chose the retrieval + tool, `llm.reply` chose to stop). The `RETRIEVER` and `TOOL` spans are the observations the agent conditioned on for the next step. The `GUARDRAIL` span is a post-filter, not an agent step — trajectory scorers ignore it as an action but read it as evidence for safety-family criteria (mod-108).

A trajectory scorer takes this sub-tree as input. It does not need the raw model API; the trace is a lossless-enough record to score against, because mod-102's schema (`llm.input_messages.*`, `llm.output_messages.*`, `llm.tool_calls.*`, `tool.name`, `input.value`, `output.value`, `llm.token_count.*`, span duration, `app.cost_usd`) carries every field a scorer needs. That property — *trajectory scoring runs off the trace, not off a re-execution* — is what makes offline replay (chapter 07) and CI gating (mod-106) possible.

### The five trajectory questions

Every trajectory scorer built in this module must answer these five on the trajectory record alone, without re-running the agent and without human inspection:

1. **Did the agent take the *right* steps?** — did each tool call use the right *name*, with the right *arguments*, in the right *order*, with dependencies satisfied? Chapter 02 defines the per-step scorers for name / arguments / ordering.
2. **Did the agent land on the *right* final answer?** — is the terminal LLM output correct against the reference (exact, partial, or judged)? Chapter 03 defines the final-answer rubric.
3. **Did the agent do it inside the *allowed budgets*?** — was `max_steps`, `max_tool_calls`, `time_limit`, `cost_usd` respected? Chapter 04 defines budgets and the rollout gates that consume them.
4. **If it failed, can we *replay* the failure deterministically?** — can we reconstruct the same trajectory from the trace + a fixed seed + a recorded tool-response table, so the fix is bisectable? Chapter 07 defines the replay contract.
5. **When the trajectory is *partly right*, how much credit does it deserve?** — a partial-credit rubric that assigns numeric credit to trajectories that got some steps right and some wrong. Chapter 03 defines the rubric.

If a scorer cannot answer one of these five, it is under-specified. The verdict a scorer emits at the end of the module is a JSON record with at least these five fields plus per-step detail; mod-106 gates CI on the fields and mod-107 attaches the record back to the trace as an `EVALUATOR` span (mod-102 schema).

### How trajectory eval differs from final-answer / single-turn eval

Trajectory evaluation is a superset of final-answer evaluation on six dimensions. The table is what to hand a reviewer who asks "why can't we just judge the reply?":

| Aspect | Single-turn / final-answer eval | Trajectory eval |
|---|---|---|
| Signal locality | One score per interaction, attached to the reply | One score per step plus a trajectory-level roll-up |
| Credit assignment | Reply is right or wrong; the *why* is opaque | Attribution to a specific step: wrong tool at step 2, right final answer despite wrong path |
| Budget attribution | Aggregate cost / latency per session; no per-step breakdown | Cost and latency per step; blame lands on the LLM span or the TOOL span that caused the blow-up |
| Retry visibility | Retries fold into one final answer; invisible | Each retry is its own step in the trajectory; retry count is a first-class SLI (chapter 04) |
| Cost / latency per step | Roll-up only; the p95 is a session number | Per-step; a slow `retriever.docs.search` shows up as its own regression signal |
| Gameability | An agent that "shortcuts" to the answer by skipping required steps still passes | Ordering / dependency scorers (chapter 02) fail the shortcut even when the answer is right |

The last row is the load-bearing one. An agent that returns the correct reply by fabricating a plausible-looking answer without calling the required verification tool passes a final-answer scorer and fails a trajectory scorer. That difference is why regulated surfaces (financial, medical, ticketing with side effects) need trajectory eval; a single-turn judge cannot see the missing step.

### How trajectory eval differs from leaderboard-shaped agent eval

Public agent benchmarks — SWE-bench (<https://www.swebench.com/>), WebArena (<https://webarena.dev/>), GAIA (<https://arxiv.org/abs/2311.12983>), τ-bench (<https://arxiv.org/abs/2406.12045>), Inspect's own bundled evals (<https://inspect.aisi.org.uk/evals/>) — publish a single leaderboard number per (model, harness). That number is useful as a **shape template** — it tells you the eval community has agreed on a task schema, a scoring function, and a reference trajectory format — but it is not the number you ship on.

Mod-101 chapter 05 draws the general line between leaderboard eval and product eval; for trajectories the same line applies. A public benchmark measures *the model's capability on a stylised task family under a fixed harness*. A product trajectory scorer measures *your surface's actual production trajectories against your product's SLOs, on your traffic distribution, with your tools, in your codebase's version of the harness*. Chapter 06 walks the mapping: take a public benchmark's task schema and scoring shape as a template, and re-express your product's trajectory eval in that shape so you can reuse the harness (chapter 05's Inspect setup, in particular) without adopting the leaderboard's task distribution.

The failure mode this section is calling out: a team that ships on "we're at 62% on τ-bench" is measuring the model, not the surface. Trajectory eval at application altitude has to be on your surface's trajectories.

### Three failure modes trajectory eval catches that end-state eval misses

**(a) Right-answer / wrong-tool ("golden shortcut").** The agent produces a correct final reply by calling a tool that returns the answer directly but that it was not authorised to call. Example: the support agent's plan required a `docs.search` retrieval to ground the reply in current policy; instead the model called an internal `sql_query` tool it had ambient access to and read the raw `orders` table. The final reply about shipment status is factually correct, so a final-answer judge scores it 5/5. A trajectory scorer with a tool-name-per-step scorer (chapter 02) flags step 2 as `expected=docs.search, actual=sql_query, verdict=wrong_tool`. This is the class that regulated surfaces care about most: the ambient-permissions blast radius is invisible until you score the step, not the reply.

**(b) Right-tool / wrong-order (dependency violated).** The agent calls all the right tools but in an order that violates a data dependency. Example: the agent calls `open_ticket` *before* `docs.search`, so the ticket body cites no policy and has to be manually edited by a support rep after the fact. The final reply to the user is unchanged ("ticket #1247 has been opened"), so a final-answer judge scores it 5/5. An ordering scorer (chapter 02) reads the trajectory, checks the DAG `docs.search → open_ticket`, and flags the trajectory as `ordering_violation`. The cost of the fix is downstream operator time, which end-state eval will never see.

**(c) Right-final / catastrophic budget (10× cost blow-up).** The agent lands on a correct answer after 11 tool calls, three of which are retries against a flaky CRM endpoint and two of which are redundant retrievals over the same query. The reply is correct; a final-answer judge scores it 5/5. A budget scorer (chapter 04) reads the trajectory's `app.cost_usd` roll-up, compares it to the surface's illustrative $0.02 / session ceiling, and flags `cost_over_budget: 0.219 vs 0.020`. In an online rollout gate this trajectory would block a canary promotion even though the answer was right. This is the class mod-111 (cost/latency/quality trade-off) rolls up to at program altitude.

### The scorer, budget, replay triad

The module's chapter arc lays out three artefacts that together make trajectory eval production-grade. Chapters 02, 03 build the **scorer**: per-step name / argument / ordering scorers plus a final-answer + partial-credit rubric that emit a structured verdict on any trajectory. Chapter 04 builds the **budget**: cost, latency, step-count, retry limits, and the rollout gates that consume them (mod-106 wires those into CI, mod-107 into the online loop). Chapter 07 builds the **replay**: a deterministic post-mortem workflow that reconstructs a trajectory from its trace so a failure can be bisected and fixed. Chapters 05, 06 give you the harness (Inspect) and the shape templates (public benchmarks) to run the triad against. Every chapter after this one is an implementation of one edge of that triad.

## Example — a worked bad trajectory

The support-agent surface from mod-102 chapter 01 has an illustrative SLO ledger: `max_steps=5`, `max_tool_calls=4`, `time_limit=8s`, `cost_usd_ceiling=0.02`, `retries_per_tool=1`. (Illustrative numbers; a real surface derives these from mod-101's SLI table and mod-111's cost model.) Consider the following production trajectory for the user input `"Where is my order 12345?"`:

```
trace c31f7a…
└─ agent.support_agent                    [AGENT, app.cost_usd=0.219, duration=14.2s]
   ├─ llm.plan                            [LLM, tokens=876, cost=0.0009, 0.5s]
   ├─ tool.sql_query                      [TOOL, args={query: "SELECT * FROM orders WHERE id=12345"},
   │                                            status=OK, 0.7s]                          # (a)
   ├─ llm.plan.2                          [LLM, tokens=921, cost=0.0011, 0.6s]
   ├─ retriever.docs.search               [RETRIEVER, query="order 12345 status", top_k=3, 0.4s]
   ├─ retriever.docs.search.2             [RETRIEVER, query="order 12345 shipping", top_k=3, 0.4s]  # (redundant)
   ├─ llm.plan.3                          [LLM, tokens=1104, cost=0.0014, 0.7s]
   ├─ tool.crm_lookup                     [TOOL, status=Error, error="502 upstream", 2.1s]  # (b)
   ├─ tool.crm_lookup.retry.1             [TOOL, status=Error, error="502 upstream", 2.0s]  # (b)
   ├─ tool.crm_lookup.retry.2             [TOOL, status=OK, 1.9s]                            # (b)
   ├─ llm.plan.4                          [LLM, tokens=1288, cost=0.0018, 0.8s]
   ├─ tool.open_ticket                    [TOOL, args={order_id: 12345,
   │                                            reason: "no_shipping_update"}, status=OK, 0.6s]   # (c)
   ├─ llm.reply                           [LLM, tokens=1421, cost=0.0021, 0.9s,
   │                                            output="Ticket #1247 opened; ETA in 2 days."]
   └─ guardrail.output_policy             [GUARDRAIL, verdict=allow]
```

The final reply is factually correct. The user is told a ticket is open and given an ETA. A final-answer judge scores it 5/5. The trajectory, however, is a mess. What each chapter's scorer flags:

- **Chapter 02 (per-step tool-call scoring)** flags step (a): the expected tool at that point in the plan DAG is `docs.search`, not `sql_query` — golden-shortcut, and a permissions violation to boot. It also flags the ordering: `open_ticket` at step (c) fires *after* `docs.search`, which is correct order, but before the retry-exhausted `crm_lookup` chain (b) — the plan's DAG required `crm_lookup` to succeed before `open_ticket`, and it did, but at a cost the ordering scorer will surface for review.
- **Chapter 03 (final-answer + partial-credit rubric)** scores the final reply as *content-correct* but assigns partial credit against the trajectory rubric: the reply cites an ETA that came from `sql_query` rather than the authoritative `docs.search` policy doc, so `groundedness_credit=0.3` even though `answer_credit=1.0`. The composite trajectory score is well below the surface's SLO on the rubric.
- **Chapter 04 (budgets and rollout gates)** flags four budget violations: `tool_calls=6 vs 4`, `steps=8 vs 5`, `duration=14.2s vs 8s`, `cost=0.219 vs 0.020`. Each is illustrative; each individually would block a canary promotion under mod-106's gate logic.
- **Chapter 07 (deterministic replay and post-mortem)** takes the trace, extracts the model inputs and the tool-response table (including the two 502 returns from `crm_lookup`), and replays the trajectory offline. The replay reproduces the retry storm deterministically, which lets the on-call bisect between "the CRM endpoint was flaky" (infra), "the retry budget was too generous" (harness config, chapter 04), and "the planner picked `sql_query` because the tool description was ambiguous" (prompt fix). Without the replay, the fix is a guess.

Chapters 05 and 06 do not flag anything on this trajectory directly — chapter 05 is the *harness* the scorers run in, and chapter 06 is the *shape template* you cross-reference against a public benchmark to sanity-check the rubric. Both are supporting infrastructure for the triad.

## Summary

- A **trajectory** is the ordered sequence of steps in one agent run, from initial input to terminal state. It is the unit of evaluation at application altitude once a surface is agentic; the final answer alone is insufficient.
- Reuse the OpenAI / Anthropic tool-use vocabulary and the OpenInference span-kind vocabulary — *step, action, observation, tool-call, tool-return, terminal state, sub-trajectory, budget*. Do not invent alternatives.
- A trajectory maps to a trace as the sub-tree under an `AGENT` span, with alternating `LLM` (action selection) and `TOOL` / `RETRIEVER` (observation) child spans and a terminal `LLM`. Trajectory scorers read the trace, not a re-execution.
- Every scorer this module builds must answer five questions on the trajectory record alone: right steps, right final answer, inside budget, replayable, and how much partial credit.
- Trajectory eval differs from final-answer eval on six dimensions — signal locality, credit assignment, budget attribution, retry visibility, per-step cost / latency, and gameability. The last is what makes it non-optional for regulated surfaces.
- Public agent benchmarks (SWE-bench, WebArena, GAIA, τ-bench) are shape templates, not product SLOs; chapter 06 walks the mapping.
- Three failure classes trajectory eval catches that end-state eval misses: right-answer / wrong-tool (golden shortcut), right-tool / wrong-order (dependency violation), right-final / catastrophic budget (cost or latency blow-up).
- The module builds a **scorer, budget, replay** triad: chapters 02–03 (scorer), chapter 04 (budget), chapter 07 (replay); chapters 05 (Inspect) and 06 (shape templates) are the harness and reference.

Chapter 02 defines the per-step tool-call scorers — name, arguments, ordering — that answer question 1.
