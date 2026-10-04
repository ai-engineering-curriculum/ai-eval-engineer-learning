# Inspect: The Agent Harness the Scorers Run In

## Motivation

Chapters 02–04 defined the scorers — per-step, final-answer, partial-credit rubric, budgets — as if they could be plugged into any runner. In practice, a scorer is only as portable as the harness it runs in: a scorer that assumes a specific trajectory shape, that owns its own sandbox, or that reinvents retry logic will not survive a hand-off to CI, to the online loop, or to a peer team. This chapter adopts the UK AI Security Institute's **Inspect** framework (<https://inspect.aisi.org.uk/>) as the reference harness for the rest of the module. The point is not that Inspect is the only choice — the Python agent-eval ecosystem has several — but that it is open-source, actively maintained, OpenInference-friendly, and already structured around the primitives this module needs: `Task`, `Sample`, `Solver`, `Scorer`, `Tool`, and first-class `message_limit` / `token_limit` / `time_limit` / `working_limit` controls (<https://inspect.aisi.org.uk/errors_and_limits.html>).

Adopting a harness has three concrete consequences for the scorer. First, the trajectory shape the scorer reads is now the harness's trajectory shape, not an ad-hoc dict. Second, the harness owns retries, limits, and sandboxing — the scorer stops reinventing them. Third, the harness has a logging and viewer story the scorer plugs into; a per-step verdict rendered next to the trace in the viewer is a different artefact from a per-step verdict dumped to stdout.

This chapter walks the mapping from the scorers of chapters 02–04 onto Inspect's primitives and establishes the file layout, decorators, and conventions the module's exercises assume.

## Core concepts

### The Inspect primitives

Five primitives carry the vocabulary. Each has a documented page on <https://inspect.aisi.org.uk/>; this chapter summarises the relationships rather than the full API surface. Read the docs from the URL — they move faster than any textbook can keep up.

**`Task`** — a benchmark unit of work. One Task bundles a dataset of `Sample`s, a `Solver` (how the agent is driven), and one or more `Scorer`s. The task is what `inspect eval` runs. The `Task` is the right place to pin the model, the `message_limit` / `token_limit` / `time_limit`, and the Solver / Scorer composition. See <https://inspect.aisi.org.uk/tasks.html>.

**`Sample`** — one input to the Task. A Sample carries at minimum an `input` (the user turn or system prompt the agent starts from) and a `target` (the reference answer or spec the Scorer compares against). For trajectory eval the Sample also carries the reference trajectory from chapter 02 (DAG, programmatic predicate, or strict sequence) in a dedicated metadata field; the convention this module uses is `metadata.reference_trajectory` and `metadata.task_family`.

**`Solver`** — a function that advances the `TaskState`. Solvers compose: a planning solver followed by a tool-execution solver followed by a reply solver is a legitimate pipeline. The Solver vocabulary is where the agent *runs*; the Scorer vocabulary is where it is *judged*. See <https://inspect.aisi.org.uk/solvers.html> and <https://inspect.aisi.org.uk/agents.html>.

**`Scorer`** — a function that reads the final `TaskState` and returns a `Score` with a value plus explanation. Scorers compose too: a per-step scorer followed by a budget scorer followed by a rubric scorer is the shape this module produces. See <https://inspect.aisi.org.uk/scorers.html>.

**`Tool`** — a function the agent can call. In Inspect, tools are typed Python functions with a decorator that advertises them to the model; the harness handles the model-API tool-use wrapping. The Tool vocabulary covers both free in-house tools (`docs.search`, `open_ticket`) and sandboxed ones (`bash`, `python.exec`). See <https://inspect.aisi.org.uk/tools.html>.

### The `TaskState` is the trajectory

Inspect's `TaskState` carries the message list (`messages`), the model output (`output`), any attached metadata, and the current `completed` status. The agent-loop is a sequence of state transforms: a Solver reads a `TaskState`, calls the model, optionally executes tool calls, and returns an updated `TaskState`. The *trajectory* this module scores is exactly this state at termination — the messages, the tool-call history, the token counts, and the timing.

Three practical consequences:

- **No separate "trajectory object".** The chapter-02 scorer reads `state.messages` and the tool-call history embedded in it. It does not need a bespoke trajectory type; Inspect's state *is* the trajectory.
- **Spans are a mirror, not the source.** Inspect emits a trace of its own (and the OpenInference auto-instrumentor mirrors it onto an OTel collector; see mod-102 chapter 03). The scorer running inside Inspect reads `TaskState`; the scorer running on a stored trace (offline, after the fact) reads mod-102's AGENT sub-tree. The two views are isomorphic by design.
- **The Scorer runs *after* the Solver.** Scorers do not re-execute the agent; they inspect the terminal state. That is what makes chapter 07's deterministic replay tractable — the Scorer never needs the model, only the state.

### Mapping the module's scorers onto Inspect scorers

Each scorer in chapters 02–04 maps onto one Inspect `Scorer` function. The composition rule: all four scorers run on every trajectory; their outputs are aggregated by a top-level rubric scorer that applies the chapter-03 weights and floors and emits the per-trajectory verdict JSON.

```
┌──────────────────────────┐
│       inspect eval        │
└────────────┬──────────────┘
             │
     ┌───────▼────────┐
     │     Task       │   dataset of Samples (one per task-family case)
     │   solver=…     │
     │   scorer=…     │
     └───────┬────────┘
             │
       ┌─────▼─────┐
       │   Solver  │   drives the agent to termination
       └─────┬─────┘
             │  final TaskState (= trajectory)
       ┌─────▼───────────────────────────────────────────┐
       │ Scorer stack                                    │
       │   per_step_scorer   (chapter 02)                │
       │   final_answer_scorer (chapter 03)              │
       │   budget_scorer     (chapter 04)                │
       │   rubric_scorer     (chapter 03, roll-up)       │
       └─────┬───────────────────────────────────────────┘
             │
       ┌─────▼─────┐
       │   Score   │   { value: 0..1, metadata: {axes, flagged_verdicts, per_step, …} }
       └───────────┘
```

Three conventions keep the stack composable:

- **Each scorer is pure.** It reads `TaskState` and the Sample's reference metadata; it does not mutate state. Pure scorers can be re-ordered, re-run, and dropped from the stack without affecting any other scorer.
- **The rubric scorer runs last.** It reads the metadata emitted by the earlier scorers and performs the weighted sum and floor check. It is the only scorer that produces the final `Score.value`.
- **Store the full breakdown in `Score.metadata`.** Inspect's `Score` already has a `value` and an `explanation`; store the per-axis vector, the `flagged_verdicts`, the attribution block, and the method labels in `metadata`. The Inspect log viewer renders this; mod-110's eval-data platform indexes on it.

### Choosing a Solver

Inspect ships several agent-loop solvers. The one most relevant to this module is a tool-using solver that drives the model until it stops calling tools or a limit is hit — the Inspect docs cover the current built-ins and recommended agent patterns at <https://inspect.aisi.org.uk/agents.html>. <!-- needs-research: on the next cycle, confirm the exact names of Inspect's built-in agent solvers (e.g., `basic_agent`, `react`, `generate` + `use_tools`) and update the citation. -->

Three selection rules:

- **Pick the Solver that matches your product's agent loop.** If your production agent is a ReAct-style loop (think-act-observe), use the Inspect solver that implements that loop. If your production agent is a plan-then-execute flow, use the solver that matches it. The point of trajectory eval is to score the *same loop you ship*; a solver mismatch produces trajectories your production never would.
- **Pin the model inside the Task.** `Task(... model=...)` or an equivalent setter. Pinning is a mod-106 CI requirement; the trajectory you score tomorrow must be the trajectory the same model produced today, modulo deterministic sampling (chapter 07).
- **Keep tool list minimal.** The Tool list the Solver sees is the Tool list the agent is permitted to use. A loose tool list is what makes "unauthorised_tool" verdicts (chapter 02) show up on a trajectory that should never have had the option. Make the authorised set explicit per Task.

### Authoring tools

Inspect's `Tool` decorator wraps a Python function so the harness can advertise it to the model and execute it on tool calls. Three authoring rules this module leans on:

- **Keep tool schemas tight.** The model's chance of calling the tool correctly scales with the clarity of the typed signature. Prefer `order_id: int` over `order_id: str`; prefer enums over free-form strings for named-set arguments; add docstrings that the harness surfaces in the tool description.
- **Return structured content.** The Tool's return value becomes the next observation. A JSON-shaped return is both easier for the next model turn to use and easier for the chapter-02 argument matcher (Level 2 exact-value) to score. Stringify only at the model boundary; keep the Python object around for the scorer.
- **Keep side effects behind a sandbox.** For tools with real side effects (`send_email`, `pay_invoice`), always route through an Inspect sandbox or a stubbed replacement in the eval harness. The sandbox is also what makes chapter 07's deterministic replay possible — a `send_email` that actually hits SMTP cannot be replayed.

### The dataset

Chapter 02's "reference trajectory per task family" lives on each `Sample`. The convention this module uses:

```python
Sample(
    input="Where is my order 12345?",
    target="Ticket #<N> opened; ETA in <2 days>.",   # final-answer reference
    metadata={
        "task_family":          "order_status_not_shipped_policy",
        "reference_trajectory": {
            "form": "DAG",
            "nodes": ["docs.search", "crm.lookup", "open_ticket"],
            "edges": [
                ["docs.search", "open_ticket"],
                ["crm.lookup",  "open_ticket"],
            ],
            "required_set": ["docs.search", "open_ticket"],
            "allowed_set":  ["docs.search", "crm.lookup", "open_ticket"],
        },
        "rubric_version":       "v1.4",
        "reference_version":    "v3.2",
        "slo": {
            "cost_usd_ceiling": 0.02,
            "time_limit_s":     8,
            "max_steps":        5,
            "max_tool_calls":   4,
            "max_retries_per_tool": 1,
        },
    },
)
```

Three invariants on the dataset:

- **One Sample per branch of the task family.** Chapter 02's four-branch rule — found+shipped, found+not-shipped+policy, found+not-shipped+no-policy, not-found — is one Sample each. More Samples per family is useful for cross-cutting cohorts; it does not substitute for branch coverage.
- **The dataset is versioned.** The eval-data platform (mod-110) owns the versioning. Each `inspect eval` run records the dataset version in the log; mod-106's CI gate reads it.
- **Load the dataset from a store, not from a Python literal.** The literal above is for illustration. Production datasets live in object storage or a database and are fetched by the Task; a change in the dataset without a version bump is a mod-110 bug.

### Running: `inspect eval` and the log

The harness entry point is `inspect eval <task>`, which produces an **eval log** — a per-run artefact containing the full `TaskState` for every Sample, the per-step model calls, tool executions, token counts, timings, and all Scorer outputs. The log is the artefact mod-106 archives for CI, mod-107 attaches to the trace back-end, and mod-109 hands to human reviewers. See <https://inspect.aisi.org.uk/log-viewer.html> for the viewer's exact UI; the browser-based log viewer is Inspect's preferred way to inspect a trajectory. <!-- needs-research: verify the log-viewer URL and the current CLI invocation on next research cycle. -->

Three practices this module adopts:

- **Fail-closed exit codes.** Configure the Task so `inspect eval` exits non-zero on any Scorer failing a floor (chapter 03). mod-106's CI gate reads the exit code.
- **Dual-emit traces.** Enable the OpenInference auto-instrumentor so Inspect's run also emits spans to the mod-102 collector. The scorer runs on the Inspect log in CI; the trace is what mod-107's online-eval loop reads.
- **Pin the Inspect version.** The Inspect version appears in the log header and should be pinned in the project's `pyproject.toml` or lockfile. A trajectory scored under Inspect v0.4.x and compared to one scored under v0.5.x may be comparing apples and oranges; the pin is part of chapter 07's replay contract.

### What *not* to put in Inspect

Three things live outside the harness even though it is tempting to pull them in:

- **The dataset owner.** Fixture authoring, versioning, and drift audits live in mod-110. A Task that embeds a dataset literal couples the harness to the fixture lifecycle; keep them separate.
- **The judge model.** When the final-answer scorer defers to a judge (chapter 03), the judge configuration lives in mod-104's config. Inspect calls out to it; it does not own it. This keeps the judge replaceable and keeps the trajectory scorer honest about which parts of the verdict are judge-graded.
- **The rollout gate.** mod-106 (CI) and mod-107 (online) own the gates. Inspect produces the per-trajectory verdict; the gate consumes it. Do not embed gate logic in a Scorer; the Scorer runs on one trajectory, the gate runs on a sample.

## Example — the support-agent Task in Inspect

A sketch of the Task module for the support agent (the imports below reflect Inspect's documented primitives; the exact decorator names evolve release-to-release — read them from the docs):

```python
# support_agent_eval/task.py  (sketch; API names per inspect.aisi.org.uk)
from inspect_ai import Task, task
from inspect_ai.dataset import Sample
from inspect_ai.scorer import scorer, Score, Target
from inspect_ai.solver import solver, TaskState
from inspect_ai.tool import tool

@tool
async def docs_search(query: str, top_k: int = 3):
    """Search the policy docs. Returns the top_k most relevant chunks."""
    ...

@tool
async def crm_lookup(order_id: int):
    """Look up the current CRM record for an order."""
    ...

@tool
async def open_ticket(order_id: int, reason: str):
    """Open a support ticket referencing an order."""
    ...

@scorer
def per_step_scorer():
    """Chapter 02: name / args / order per step."""
    async def score(state: TaskState, target: Target) -> Score:
        ref = state.metadata["reference_trajectory"]
        verdicts = compare_steps(state.messages, ref)
        return Score(value=per_step_summary(verdicts),
                     metadata={"per_step": verdicts})
    return score

@scorer
def budget_scorer():
    """Chapter 04: cost / latency / steps / retries."""
    ...

@scorer
def rubric_scorer():
    """Chapter 03: weighted sum + floors; final per-trajectory verdict."""
    async def score(state: TaskState, target: Target) -> Score:
        axes = roll_up(state.metadata["per_step_score"],
                       state.metadata["final_answer_score"],
                       state.metadata["budget_score"])
        flagged = collect_verdicts(axes)
        if any_floor_tripped(flagged, task_family=state.metadata["task_family"]):
            return Score(value=0.0,
                         metadata={"pass": False, "floor_tripped": True,
                                   "axes": axes, "flagged_verdicts": flagged})
        return Score(value=weighted_sum(axes),
                     metadata={"pass": True, "axes": axes,
                               "flagged_verdicts": flagged})
    return score

@task
def support_agent_order_status():
    return Task(
        dataset=load_dataset("support_agent/order_status/v3.2"),
        solver=react_agent(tools=[docs_search, crm_lookup, open_ticket]),  # Inspect built-in
        scorer=[per_step_scorer(), budget_scorer(), rubric_scorer()],
        message_limit=12,
        token_limit=20_000,
        time_limit=20,     # harness hard-stop; above the 8s SLO
        model="anthropic/claude-opus-4-7",
    )
```

Run:

```
$ inspect eval support_agent_eval.task:support_agent_order_status \
      --log-dir ./logs/order_status/
$ inspect view ./logs/order_status/
```

The CLI invocation writes a log; the viewer renders each Sample's trajectory next to the per-axis scorer output. In CI, a wrapper script reads the log, aggregates to pass-rate and p95s (chapter 04), and exits non-zero on floor trips or budget overshoots. mod-106 wires that wrapper into the branch-protection check.

## Common pitfalls

- **Scorer mutates state.** A Scorer that modifies `TaskState` makes downstream Scorers see different inputs than the Solver produced. Keep Scorers pure.
- **Solver that reimplements the model loop by hand.** Rebuilding the tool-use loop (parse tool calls, execute, inject result, loop) in a custom Solver instead of using Inspect's agent primitives is a maintenance load that drifts from the Inspect release cadence. Prefer the built-in agent solvers unless your production loop is genuinely non-standard.
- **One scorer that does everything.** The scorers compose intentionally: per-step, final-answer, budget, rubric. A monolithic scorer collapses the attribution the rubric exists to preserve (chapter 03).
- **Tool side effects without a sandbox.** A `send_email` tool that reaches SMTP during eval sends real email on every run. Route side-effectful tools through a sandbox or stub; wire the real tool only in production.
- **Loose tool list.** The Solver's tool list is the authorised set. A Task that exposes every tool the production surface has makes "unauthorised_tool" verdicts inherently unreachable.
- **Dataset literal in the Task module.** Couples the harness to the fixture lifecycle. Load from mod-110's dataset store.
- **No log retention policy.** Inspect's logs are the artefact mod-106 archives for CI, mod-107 attaches online, and mod-109 hands to reviewers. Decide where they live and how long they live before the first `inspect eval` run.

## Summary

- Inspect (<https://inspect.aisi.org.uk/>) is the module's reference agent harness: open-source, actively maintained, OpenInference-friendly, with first-class `message_limit` / `token_limit` / `time_limit` / `working_limit`.
- Five primitives: `Task` bundles everything; `Sample` is one input + reference; `Solver` drives the agent; `Scorer` reads terminal state and returns a `Score`; `Tool` is a decorated function the agent can call.
- The `TaskState` *is* the trajectory. The scorers from chapters 02–04 are pure functions over `TaskState`; the rubric scorer runs last and emits the per-trajectory verdict.
- Pin the model, the tool list, the dataset version, and the Inspect version inside the Task. These pins are part of chapter 07's deterministic-replay contract.
- Keep what Inspect does *not* own outside Inspect: fixture lifecycle (mod-110), judge configuration (mod-104), rollout gates (mod-106 / mod-107).
- The eval log is the hand-off artefact: CI consumes exit codes, the online loop consumes attached traces, human review consumes the browser-rendered trajectory.
- Pitfalls — mutating state in a Scorer, hand-rolling the agent loop, monolithic scorers, side-effectful tools without a sandbox, loose tool lists, literal datasets, no log retention — are all solvable by applying the five primitives as they are documented.

Chapter 06 covers the public agent benchmarks (SWE-bench Verified, τ-bench, WebArena, GAIA, AgentBench, BFCL, ToolBench) as *shape templates* for internal agent suites: what to borrow, what to leave on the leaderboard.
