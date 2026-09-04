# Multi-Runner Orchestration Behind One Plane: The Adapter Contract

## Motivation

A single org almost never lands on a single eval runner. The reasons are structural, not fashion-driven:

- **Different eval shapes fit different runners.** RAGAS is idiomatic for RAG faithfulness / context-precision / context-recall metrics; DeepEval is a strong fit for regression-suite-per-metric workflows in Python; Promptfoo is the shortest path for prompt-comparison matrices in YAML; Braintrust and Weave and Langfuse each shine on a hosted UI with trace linking; a custom Python evaluator is the only fit for a bespoke business metric that no library ships.
- **Different teams have different vendor accounts already.** Team A has a Braintrust project, Team B has a Weave workspace, Team C uses Langfuse OSS. Consolidating everyone onto one is a political problem the eval team should not spend six months on.
- **The best-of-breed shifts.** The runner that was the strongest in 2025 is not necessarily the strongest in 2027. A platform that has locked in one runner has to be re-written when the org outgrows it; a platform that abstracts over runners survives the swap.

If each runner has its own SDK, its own result shape, its own cost accounting, and its own UI, the eval program is *N* programs that happen to share a Slack channel. The plane's job is to publish one contract every runner is wrapped behind, so that a request in — `EvalRunSpec` — produces a result out — `EvalResultRow` — regardless of which runner picked it up.

This chapter fixes the contract, walks the adapter interface, and notes the specific integration surface for each of the runners the module lists (Promptfoo, DeepEval, RAGAS, Braintrust, W&B Weave, Langfuse) plus a custom-evaluator adapter shape.

## Core concepts

### The control-plane / data-plane split

Chapter 01 named the analogy; this chapter operationalises it.

- The **plane** (control plane) — the code the eval-data-platform team writes — is responsible for: accepting an `EvalRunSpec`, resolving lineage keys, checking the quota, enqueueing the run, dispatching to a runner adapter, tracking retries and idempotency, collecting results, normalising them into `eval_results`, and emitting the OpenLineage `RunEvent`.
- The **runner** (data plane) — Promptfoo / DeepEval / RAGAS / Braintrust / Weave / Langfuse / custom — is responsible for: taking the cases, calling the target model, calling the judge, computing per-case scores, and returning them. The runner does *not* touch the schema, the quota, the cost centre, or the SLO.
- The **runner adapter** — a small module the eval team writes per runner — bridges the two: takes an `EvalRunSpec` from the plane, translates it into whatever the runner needs, invokes the runner, collects the runner's native output, translates it back into `EvalResultRow`s, and reports cost / latency to the plane.

The split has one crisp implication: the plane never imports a runner's SDK directly. Only the adapter does. This is what makes swapping a runner a *local* change and not a plane-wide refactor.

### The `EvalRunSpec` — what the plane sends

The `EvalRunSpec` is the plane's request shape. It is small enough to serialise as JSON, wide enough to carry the whole lineage key set, and stable across runner adapters.

```json
{
  "spec_version": "1.0",
  "run_id": "run_01JAV8...",
  "spec_hash": "sha256:1f2e...",
  "cost_centre": "team-support/project-bot",
  "triggered_by": "pr_gate",
  "triggered_at": "2026-09-04T14:00:00Z",
  "priority": "class_a",
  "surface": "support_bot",
  "target": {
    "kind": "http",
    "endpoint": "https://api.example.com/answer",
    "model_snapshot": "gpt-4o-2024-08-06",
    "prompt_hash": "sha256:beef...",
    "chain_hash": "sha256:cafe...",
    "retriever_index_hash": "sha256:1234...",
    "decoding_config_hash": "sha256:5678...",
    "seed": 42
  },
  "eval_set": {
    "eval_set_id": "support_bot_golden",
    "eval_set_hash": "sha256:9abc..."
  },
  "rubrics": [
    {"rubric_id": "faithfulness_v3",   "rubric_hash": "sha256:aaaa...", "judge_model_snapshot": "claude-3-5-sonnet-20241022"},
    {"rubric_id": "safety_refusal_v2", "rubric_hash": "sha256:bbbb...", "judge_model_snapshot": "gpt-4o-2024-08-06"}
  ],
  "budget": {"soft_usd": 5.00, "hard_usd": 20.00, "hard_walltime_s": 900},
  "artefact_sinks": ["eval_results", "openlineage", "vendor_native"],
  "idempotency_key": "pr:1234:commit:abc123:eval:support_bot_golden:sha256:aaaa..."
}
```

Every field is required. Two are load-bearing enough to name explicitly:

- **`idempotency_key`.** A stable string the plane derives from the trigger context. Two triggers of the same PR-commit-eval combination produce the same key; the plane deduplicates on it. Chapter 05 covers the retry contract that hangs off this key.
- **`spec_hash`.** SHA-256 over the canonicalised spec (excluding `run_id`, `triggered_at`, `idempotency_key`). Two runs with the same `spec_hash` are semantically the same run; the reproducibility check joins on it.

### The `EvalResultRow` — what the runner returns

Every runner emits one row per (case × metric). The row shape matches the `eval_results` table from chapter 02, with an added `runner_native_row_id` for round-tripping to the runner's UI.

```json
{
  "run_id": "run_01JAV8...",
  "case_id": "CASE-support-billing-refund-2026-03-12",
  "metric": "faithfulness",
  "score": 0.87,
  "verdict": null,
  "score_confidence": 0.91,
  "trace_id": "5b7e...",
  "session_id": null,
  "model_snapshot": "gpt-4o-2024-08-06",
  "prompt_hash": "sha256:beef...",
  "chain_hash": "sha256:cafe...",
  "retriever_index_hash": "sha256:1234...",
  "decoding_config_hash": "sha256:5678...",
  "seed": 42,
  "eval_set_hash": "sha256:9abc...",
  "rubric_hash": "sha256:aaaa...",
  "judge_model_snapshot": "claude-3-5-sonnet-20241022",
  "source": "offline_gate",
  "scored_at": "2026-09-04T14:03:12Z",
  "runner_native_row_id": "brt://experiments/exp_123/rows/row_456",
  "judgement": {
    "judge_tier": "mid",
    "judge_prompt_hash": "sha256:cccc...",
    "judge_input_preview": "You are evaluating faithfulness...",
    "judge_output_preview": "0.87 — the answer is supported by the retrieved chunk...",
    "cost_usd": 0.0042,
    "latency_ms": 812
  }
}
```

Two rules:

- **Every runner adapter emits the full row.** Fields the runner does not natively record are filled in by the adapter from the `EvalRunSpec` (all lineage keys) or from the runner's cost / latency accounting (the `judgement` block).
- **`runner_native_row_id` is a URI.** `brt://` for Braintrust, `wandb://` for Weave, `lfs://` for Langfuse, `pf://` for Promptfoo local, `deepeval://` for DeepEval, `custom://<runner>/...` for a bespoke evaluator. This is what lets the mod-107 dashboard link back to the runner's native trace UI without re-implementing it.

### The runner-adapter interface

The interface every adapter implements. Four methods; no more.

```python
class RunnerAdapter(Protocol):
    name: str  # "promptfoo" | "deepeval" | "ragas" | "braintrust" | "weave" | "langfuse" | "custom:<name>"
    supported_metric_families: set[str]  # e.g., {"faithfulness", "groundedness", "context_precision"}

    def plan(self, spec: EvalRunSpec) -> PlannedRun:
        """Validate the spec against the runner's capabilities.
        Return a PlannedRun with an estimated cost, an estimated walltime,
        and any warnings (e.g., a metric the runner will approximate rather than
        compute exactly). Never calls out to the model or the judge.
        """

    def run(self, plan: PlannedRun) -> RunHandle:
        """Kick off the run. Non-blocking. Returns a handle the plane polls."""

    def stream_results(self, handle: RunHandle) -> Iterable[EvalResultRow]:
        """Yield EvalResultRow objects as the runner produces them.
        Every row conforms to the schema above. The adapter is responsible for
        filling in lineage fields from the spec.
        """

    def cost_report(self, handle: RunHandle) -> CostBreakdown:
        """Return the runner's actual cost and latency accounting, itemised by
        (metric, judge_model_snapshot, judge_tier).
        Called once the run terminates.
        """
```

That is the whole contract. Anything beyond these four methods lives in the plane, not the adapter.

Two secondary responsibilities every adapter also owns:

- **Bidirectional sync of the eval set.** Before `run`, the adapter creates or updates the runner's native dataset from `eval_cases` and tags it with the `eval_set_hash`. After `run`, the adapter reports any *new* cases the runner encountered (rare — usually a runner has added a case in its UI) as proposed contributions per chapter 03's flow.
- **Cache participation.** The adapter reads the plane's cache before invoking the runner. Cache key: `(rubric_hash, judge_model_snapshot, case_hash, target_output_hash)`. On a hit, the adapter emits the cached `EvalResultRow` without paying the judge; the `cost_usd` is zero and `cache_hit=true` is recorded on the row.

### Runner-by-runner integration notes

The specific integration surface each runner exposes. Read from the runner's docs before wiring — the surfaces evolve.

**Promptfoo.** YAML-driven eval runner; CLI-first. `promptfoo eval -c <config>.yaml --output <path>` is the primary run command; the config declares `providers`, `prompts`, `tests`, and `assertions`. The adapter's job: translate the `EvalRunSpec` and the `eval_cases` into a `promptfoo` YAML, invoke the CLI (or the Node/Python API), and parse the JSON output into `EvalResultRow`s. Well-suited to prompt-comparison matrices (multiple `providers` × multiple `prompts`) and to LLM-rubric assertions (`llm-rubric`, `model-graded-closedqa`, etc.). Cost accounting: Promptfoo reports token counts per row; the adapter maps to `cost_usd` using the plane's `pricing_snapshot`.

**DeepEval.** Python-native regression-suite runner. `deepeval test run` executes a set of `LLMTestCase`-driven metric evaluations, each returning a score, a reason, and a pass / fail against a threshold. The adapter's job: construct `LLMTestCase` objects from `eval_cases`, wire the metrics the `EvalRunSpec.rubrics` name (Deepeval ships FaithfulnessMetric, HallucinationMetric, BiasMetric, etc.; a custom Python metric implements `BaseMetric`), invoke the runner, and translate the results. Cost accounting: DeepEval reports per-metric costs; the adapter aggregates.

**RAGAS.** Python library specialised for RAG evaluation. Metrics include `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall`, `answer_similarity`. The runner takes a `Dataset` (HuggingFace-shaped: question / answer / contexts / ground_truth columns) and returns a `Result` object. The adapter's job: build the HF `Dataset` from `eval_cases` (this requires the case to carry the retrieved contexts, so the target's `chain_hash` must expose them), invoke `evaluate(...)`, and translate per-row scores. RAGAS's judge model is configurable; the adapter passes the `judge_model_snapshot` from the spec.

**Braintrust.** Hosted eval and prompt-management platform. Primary Python entry point is `braintrust.Eval(...)` (or the SDK-level `braintrust.init_experiment` for finer control); results land in a Braintrust `Experiment` with a linked UI. The adapter's job: initialise a Braintrust project and experiment tagged with the `run_id`, upload the cases as a `Dataset` (or reference an existing one keyed by `eval_set_hash`), run the eval, and stream the experiment rows back. Cost accounting: Braintrust reports per-experiment cost in the UI; the SDK exposes it via `experiment.summarize()` on modern versions. <!-- needs-research: on the next research cycle, re-verify the Braintrust Python SDK entry point names and the current cost-report call at https://www.braintrust.dev/docs — the SDK ships weekly and the class names shift. -->

**W&B Weave.** Weights & Biases's LLM evaluation surface. Primary entry point is `weave.Evaluation(dataset=..., scorers=[...])`, invoked with `evaluation.evaluate(model)`. Traces land in the Weave UI; scores are attached to trace spans. The adapter's job: build a `weave.Dataset` (or reference by name), wrap the target model as a `weave.Model`, wrap each rubric as a `weave.Scorer`, and invoke `evaluate`. Cost accounting: Weave reports token counts on each op; the adapter maps to `cost_usd`. <!-- needs-research: on the next research cycle, re-verify the current W&B Weave evaluation entry points at https://weave-docs.wandb.ai and confirm the `Scorer` / `Model` decorator conventions match the current SDK. -->

**Langfuse.** OSS trace and evaluator platform (also hosted). Two evaluation surfaces: an SDK-level `langfuse.score(...)` that attaches a score to a trace, and a UI-configurable "evaluators" surface that runs a model-graded score periodically against a dataset. The adapter's job: create a Langfuse `Dataset` from `eval_cases` (or reference one), run the evaluator (usually via the SDK for offline runs), and stream scores back via `langfuse.fetch_scores(...)`. Cost accounting: Langfuse's trace observations record token usage; the adapter aggregates. <!-- needs-research: on the next research cycle, re-verify the Langfuse Python SDK's dataset / evaluator / score fetch entry points at https://langfuse.com/docs — the SDK ships frequently and the surface has changed multiple times. -->

**Custom evaluator.** The team's own Python module: `def score(case: dict, output: dict) -> dict` returning `{score, reason, cost_usd, latency_ms}` (or the categorical `verdict` equivalent). The adapter's job: iterate cases, call the target, call `score`, and translate. No hosted surface; the run is fully in the plane's process. The custom-evaluator adapter is the *reference implementation* — the smallest working adapter, useful for testing the plane's contract before wiring any vendor.

The adapter list is not exhaustive. Trulens, Inspect, Evidently-LLM, and Arize Phoenix's own evaluator surface are all valid targets on the same interface. The chapter picks the seven the module named; the exercise wires three (one hosted, one OSS, one custom).

### Metric normalisation

Different runners score "faithfulness" differently. Promptfoo's `llm-rubric` might return `{pass: true, score: 0.87, reason: "..."}`. DeepEval's `FaithfulnessMetric` returns `{score: 0.87, threshold: 0.5, reason: "..."}`. RAGAS's `faithfulness` returns `[0.87]` for a single-row dataset. Braintrust returns whatever the scorer function returned. Weave's scorer returns a dict.

The adapter is responsible for the translation into the canonical `EvalResultRow.metric` + `score` + `verdict` triple. Two rules:

- **The metric name is canonical.** The `EvalRunSpec.rubrics[*].rubric_id` names the metric (`faithfulness_v3`). The runner's native name is a *view*; the row records the canonical name.
- **The score scale is documented in the rubric spec.** A rubric that scores 0 – 1 vs a rubric that scores 1 – 5 vs a rubric that returns PASS / FAIL are all legal, but the rubric spec (chapter 03 of mod-104) names the scale, and the runner adapter refuses to run if the runner's output does not conform. A silent rescale ("we just divide by 5") is the "one giant table" anti-pattern in disguise.

### Two integration patterns for the plane's process model

The plane can run adapters in-process (the `run` method starts a subprocess or an async task the plane monitors) or out-of-process (the `run` method enqueues a job for a worker fleet the plane orchestrates via a message bus).

- **In-process** is right for a team with ≤ ~ten runs / minute and a single-region deploy. The plane is a single service; the adapters are libraries the service imports; runs are `asyncio.Task`s or subprocesses. Easier to operate; harder to scale.
- **Out-of-process** is right for a team with cross-team traffic and multiple regions. The plane is a control-plane API; the adapters live in worker containers; runs are dispatched via a queue (RabbitMQ, SQS, Redis Streams, or the org's standard). Scales cleanly; more moving parts to operate.

The exercise defaults to in-process and notes the out-of-process migration path. Either is correct; the plane's *contract* — `EvalRunSpec` in, `EvalResultRow` out, four adapter methods — does not change.

### The `run_manifest.json` — what the plane emits back to the trigger

Every run produces a manifest the trigger (CI, online loop, scheduler) can read. This is the boundary artefact between the plane and the trigger; chapter 05 wires it into the mod-106 gate.

```json
{
  "run_id": "run_01JAV8...",
  "spec_hash": "sha256:1f2e...",
  "status": "succeeded",
  "runner": "braintrust",
  "started_at": "2026-09-04T14:00:03Z",
  "ended_at":   "2026-09-04T14:03:47Z",
  "cost_usd": 4.32,
  "cost_centre": "team-support/project-bot",
  "results_count": 512,
  "cache_hit_count": 384,
  "artefact_uris": {
    "eval_results_query": "select * from eval_results where run_id = 'run_01JAV8...'",
    "vendor_native": "https://braintrust.dev/app/eval-slice/experiments/exp_123",
    "openlineage_event": "https://openlineage.example.com/events/evt_01JAV8..."
  },
  "warnings": [
    "Runner 'braintrust' approximated metric 'context_precision' — see rubric spec footnote"
  ]
}
```

The `artefact_uris` block is what makes the run *discoverable*. Every downstream consumer (the PR gate, the dashboard, the runbook) reads the manifest and follows the links; no consumer re-derives them.

### What the adapter does *not* do

- **Own the quota.** The plane does. An adapter that enforces its own rate limit will fight with the plane and neither will win.
- **Own the retry policy.** The plane does. An adapter that retries on its own will double the request count and burn the cost centre's budget.
- **Own the schema.** Chapter 02 does. An adapter that adds a column has already lost the "one shape across runners" property.
- **Own the UI.** The runners' UIs are the UIs. The plane exposes their links in the `run_manifest.json`.

Every one of those is a boundary the exercise checks. Adapters that violate them get flagged in code review.

## Summary

- The plane is a **control plane**; the runners are the **data plane**; the **runner adapter** is the small module that bridges them. The plane never imports a runner's SDK — only the adapter does.
- The **`EvalRunSpec`** carries the full lineage key set, the cost centre, the priority, the budget, and an **`idempotency_key`**. Two triggers with the same key deduplicate to one run.
- The **`EvalResultRow`** matches chapter 02's `eval_results` shape and adds a `runner_native_row_id` URI for round-tripping to the runner's UI.
- The adapter interface is four methods — `plan`, `run`, `stream_results`, `cost_report` — plus two secondary responsibilities (bidirectional eval-set sync, cache participation).
- Seven runners are named — **Promptfoo, DeepEval, RAGAS, Braintrust, W&B Weave, Langfuse, custom**. Each has a specific integration surface; the exercise wires three.
- **Metric normalisation** happens in the adapter — the canonical name is the `rubric_id`; the runner's native name is a view; a silent score-rescale is not allowed.
- **In-process** vs **out-of-process** are both correct process models; the contract does not change; pick by scale.
- Every run emits a **`run_manifest.json`** with status, cost, results count, and the vendor-native / OpenLineage links; this is the boundary artefact the trigger reads.
- The adapter **does not** own the quota, retries, schema, or UI. Those are plane responsibilities. Boundary violations get flagged in code review.

Chapter 05 wires the plane into the mod-106 PR gate and the mod-107 online loop with quotas, rate-limits, retries, idempotency, and per-team cost accounting — the machinery that makes the plane operable under real cross-team traffic.
