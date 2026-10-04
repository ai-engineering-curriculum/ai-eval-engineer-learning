# mod-103 — Trajectory and Tool-Call Evaluation for Production Agents

**Estimated effort:** 16 hours

This is the third module of the AI Evaluation Engineer track. It takes the SLIs from `mod-101` and the traces from `mod-102` and turns them into the per-trajectory verdict the rest of the track consumes. `mod-104` (LLM-as-judge) is one implementation of a trajectory scorer; `mod-106` (CI gates) and `mod-107` (online eval) consume the per-trajectory verdict directly; `mod-111` (cost / latency / quality) rolls up the budget axis defined here; `mod-112` (program ownership) aggregates the floors and the `flagged_verdicts` vocabulary into the program-level report. Nothing in those downstream modules works if the scorer this module builds is wrong, so the shape of the verdict is deliberate.

## Learning objectives

- Design a trajectory-level scorer for a real product agent: per-step tool-call correctness (name, arguments, ordering), final-answer correctness, and a partial-credit rubric.
- Track and enforce budgets — cost per trajectory, latency per step, step count, retry rate — and gate a rollout on them.
- Use the UK AISI Inspect agent harness (solvers, scorers, tools) to author a product-shaped agent eval.
- Adopt SWE-bench Verified, τ-bench, WebArena, GAIA, AgentBench, the Berkeley Function-Calling Leaderboard, and ToolBench as *shape templates* for internal agent suites — without confusing them with the product-agent eval itself.
- Design deterministic replay for failing trajectories (recorded tool responses, seeded randomness, pinned model version) and a post-mortem workflow that uses it.

## Lecture chapters

1. [Why trajectory-level evaluation for production agents](01-why-trajectory-eval.md) — vocabulary (trajectory, step, action, observation, tool-call, terminal state, sub-trajectory, budget), the five trajectory questions, three failure modes end-state eval misses, and the **scorer / budget / replay** triad.
2. [Scoring per-step tool calls: name, arguments, ordering](02-scoring-per-step-tool-calls.md) — reference trajectory forms (strict / set / DAG / programmatic), name matching with aliases and MCP namespaces, the five-level argument-match ladder (structural → judge-graded), partial-order matchers using `graphlib.TopologicalSorter`, and the per-step verdict JSON shape.
3. [Final-answer scoring and the partial-credit rubric](03-final-answer-and-partial-credit-rubric.md) — six final-answer methods (exact / overlap / programmatic / reference-judge / reference-free judge / structured verifier), the groundedness axis, pre-registered weights with per-verdict floors, the verdict taxonomy, and per-slice aggregation.
4. [Trajectory budgets and rollout gates](04-budgets-and-rollout-gates.md) — the four budget families (cost / latency / steps / retries), the `budget_credit` computation with worst-family-wins, the attribution block (`cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool`), and the offline (`mod-106`) and online (`mod-107`) rollout gates.
5. [Inspect: the agent harness the scorers run in](05-inspect-agent-harness.md) — Inspect's five primitives (`Task`, `Sample`, `Solver`, `Scorer`, `Tool`), mapping the module's scorers onto a pure-Scorer stack with a final rubric scorer, dataset conventions, log viewer, and what *not* to put in Inspect.
6. [Public agent benchmarks as shape templates](06-public-benchmarks-as-shape-templates.md) — SWE-bench Verified, τ-bench, WebArena, GAIA, AgentBench, BFCL, and ToolBench — what to borrow, what to replace, what to leave on the leaderboard; a unified mapping table from each benchmark to the scorer stack.
7. [Deterministic replay and the post-mortem workflow](07-deterministic-replay-and-post-mortem.md) — the six pins (model, params, messages, tool returns, environment, harness), the recorded tool-response table, the Inspect replay harness, the six-step post-mortem (**lift, pin, reproduce, bisect, fix, learn**), and when replay is impossible.

## Exercises

Each exercise builds on the one before it. Do them in order and keep the outputs; the artefacts are the deliverables `mod-106`, `mod-107`, `mod-109`, and `mod-110` will assume you have.

1. [Trajectory scorer with tool call and budget](exercises/exercise-01-trajectory-scorer-with-tool-call-and-budget.md) — build the per-step scorer (name, args, order) and the budget scorer against a captured trajectory; emit the per-step and budget verdict JSON.
2. [Partial credit rubric for a product agent](exercises/exercise-02-partial-credit-rubric-for-product-agent.md) — assemble the chapter-03 rubric with pre-registered weights, floors, and per-slice aggregation; roll per-step and budget verdicts into a per-trajectory verdict.
3. [Inspect agent harness: solver + scorer](exercises/exercise-03-inspect-agent-harness-solver-scorer.md) — wrap the scorers from exercises 1–2 as Inspect `Scorer`s inside a `Task` with a real `Solver`; produce an eval log and read it in the Inspect viewer.
4. [Public benchmark shape-template audit](exercises/exercise-04-public-benchmark-shape-template-audit.md) — audit three public benchmarks against your surface's task family and produce a shape-template adoption plan plus a stakeholder memo.
5. [Deterministic replay for failing trajectories](exercises/exercise-05-deterministic-replay-for-failing-trajectories.md) — materialise the six-pin replay context for a failing trajectory, write the recorded tool-response table, and walk the six-step post-mortem.

## Labs and quizzes

- Labs (see [`labs/`](labs)) wire the scorer stack into Inspect end-to-end on a reference support-agent surface. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary (trajectory / step / action / observation / terminal / sub-trajectory / budget), the five trajectory questions, the four reference-trajectory forms, the five argument-match levels, and the six replay pins. Authored under the autonomous fill-in loop.

## Resources

External references — Inspect, OpenInference, OpenAI and Anthropic tool use, the seven public benchmarks, replay / snapshot tooling, and reporting-shape primary sources — are curated in [`resources.md`](resources.md).

## Where this module hands off

- LLM-as-judge rubric design and bias control, where the chapter-03 Level-5 arg matcher and the reference-free final-answer judge are deepened → [`mod-104-llm-as-judge-in-product`](../mod-104-llm-as-judge-in-product).
- RAG-triad groundedness scoring that fills the `groundedness_credit` axis of the chapter-03 rubric → [`mod-105-rag-eval-app-layer`](../mod-105-rag-eval-app-layer).
- CI gates that read the per-trajectory verdict's `pass`, `floor_tripped`, and p95 budget axes → [`mod-106-eval-gated-cicd`](../mod-106-eval-gated-cicd).
- Online-eval sampling loop that attaches the verdict back to the trace as an `EVALUATOR` span and consumes the attribution block for canary gates → [`mod-107-online-eval-and-regression`](../mod-107-online-eval-and-regression).
- Safety-family scorers that extend the chapter-03 verdict taxonomy with guardrail-shaped verdicts → [`mod-108-app-safety-and-guardrails-eval`](../mod-108-app-safety-and-guardrails-eval).
- Human review workflows fed by the chapter-07 post-mortem's annotations and the regression-fixture promotion in Step 6 → [`mod-109-human-review-workflows`](../mod-109-human-review-workflows).
- Fixture store (reference trajectories, equivalence classes, price tables, dataset versioning) → [`mod-110-eval-data-platform-slice`](../mod-110-eval-data-platform-slice).
- Cost / latency / quality trade-off dashboards consuming the four budget axes and the attribution block → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- Program-level aggregation of the `flagged_verdicts` vocabulary, the rubric versions, and the model-version retention policy that makes replay possible → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
