# exercise-03: Multi-Runner Orchestration Behind One Plane

**Estimated effort:** 4 hours

## Objective

Ship the **eval control plane** chapter 04 describes: the `EvalRunSpec` / `EvalResultRow` / `run_manifest.json` contract, the `RunnerAdapter` interface, and **three** working runner adapters — one hosted (Braintrust, W&B Weave, or Langfuse), one OSS (Promptfoo, DeepEval, or RAGAS), and the **custom** reference adapter (an in-process Python evaluator). Run **the same `EvalRunSpec`** through all three adapters and confirm that every row they produce normalises into the same `eval_results` schema from exercise-01, with every lineage key populated.

By the end of this exercise, a single spec in → three runs → three sets of normalised rows all queryable with the same SQL. The adapters do not share code beyond the interface; the plane does not import any runner's SDK; swapping a fourth runner in is a copy-paste-adapt of ~200 lines. This is the shape the chapter 06 platform-SLI dashboard and the chapter 07 delegation hand-off both rely on.

This is the surface **exercises 04 and 05** read from. Exercise 04's "runner health" dashboard tile reads your adapters' per-row cost and latency metadata; exercise 05's PR gate posts a spec through this plane and reads back the `run_manifest.json`. Build the contract carefully — downstream code assumes it is stable.

## Prerequisites

- Chapters 01, 02, and 04 of this module.
- The exercise-01 schema. Specifically, the `eval_runs`, `eval_results`, `judgements`, `pricing_snapshots`, and `eval_set_current` tables; the `canonicalisation/`, `hashing.py`, and `models.py` modules. All three adapters write into these tables.
- The exercise-02 test-case surface. The plane reads the current `eval_set_hash` for a suite from `eval_set_current`; the adapters resolve the case list from `eval_cases` filtered on that hash.
- Python 3.11+; `pydantic` v2, `httpx`, `asyncio`, a Postgres client. For the hosted-vendor adapter pick one — the SDKs below are current entry points:
  - **Braintrust** — `pip install braintrust` and docs at <https://www.braintrust.dev/docs>.
  - **W&B Weave** — `pip install weave` and docs at <https://weave-docs.wandb.ai>.
  - **Langfuse** — `pip install langfuse` and docs at <https://langfuse.com/docs>.
- For the OSS adapter pick one:
  - **Promptfoo** — `npm i -g promptfoo` and docs at <https://www.promptfoo.dev/docs>.
  - **DeepEval** — `pip install deepeval` and docs at <https://docs.confident-ai.com/>.
  - **RAGAS** — `pip install ragas` and docs at <https://docs.ragas.io/>.
- A target application the adapters can call. A simple HTTP endpoint that answers one input turn is enough — reuse the mod-105 RAG stack, the mod-103 agent, or ship a 20-line stub that echoes a canned response.
- A judge model you can call. A hosted model (OpenAI, Anthropic, Bedrock, Vertex) or a local one via Ollama is fine. The exercise uses the judge sparingly — budget `$1 – 5` of judge spend to run the demonstration end-to-end.
- Keys / credentials for the hosted runner you picked, loaded via environment variables (`BRAINTRUST_API_KEY`, `WANDB_API_KEY`, `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY`). Do not commit them.

## Set-up

1. Create `eval/platform/plane/`:

   ```
   eval/platform/plane/
   ├── config.yaml
   ├── spec/
   │   ├── eval_run_spec.py              # pydantic — the EvalRunSpec
   │   ├── eval_result_row.py            # pydantic — the EvalResultRow
   │   ├── run_manifest.py               # pydantic — the run_manifest.json
   │   └── cost_breakdown.py
   ├── adapters/
   │   ├── base.py                       # RunnerAdapter protocol
   │   ├── custom/                       # reference adapter — in-process Python
   │   │   ├── adapter.py
   │   │   └── scorer.py                 # a tiny LLM-rubric judge
   │   ├── promptfoo/                    # or deepeval/ or ragas/
   │   │   ├── adapter.py
   │   │   └── template.yaml             # promptfoo config template
   │   └── braintrust/                   # or weave/ or langfuse/
   │       └── adapter.py
   ├── control/
   │   ├── dispatch.py                   # the plane's run loop
   │   ├── cache.py                      # (rubric_hash, judge, case_hash, output_hash) cache
   │   └── normalise.py                  # metric / score / verdict normaliser
   ├── openlineage/
   │   └── emit.py                       # (reuses exercise-01's emitter)
   ├── cli/
   │   └── plane.py                      # `plane run <spec.json>` for local iteration
   ├── tests/
   │   ├── test_spec_roundtrip.py
   │   ├── test_adapter_contract.py      # every adapter implements the 4 methods and emits the full row
   │   ├── test_custom_adapter.py
   │   ├── test_oss_adapter.py
   │   ├── test_hosted_adapter.py
   │   └── test_three_runner_parity.py   # the headline test
   └── README.md
   ```

2. Fill `config.yaml`:

   ```yaml
   db:
     dsn: postgresql://eval:eval@localhost:5432/eval_platform
   target:
     kind: http
     endpoint: http://localhost:8000/answer
   adapters:
     custom:
       enabled: true
     promptfoo:
       enabled: true
       binary: /usr/local/bin/promptfoo
     braintrust:
       enabled: true
       project: eval-slice-mod110
   cache:
     backend: sqlite
     path: ./plane_cache.sqlite
     ttl_days: 30
   process_model: in_process              # out_of_process is the stretch goal
   openlineage:
     endpoint: http://localhost:5000/api/v1/lineage
   ```

3. Pick one suite from exercise-02 as the test subject — the smallest is best. A 20 – 50-case subset is enough to prove the three-runner parity; a 500-case run is a stretch goal.

4. Confirm that your target HTTP endpoint answers at least one turn end-to-end with the model snapshot you plan to pin. Record the `model_snapshot`, `prompt_hash`, `chain_hash`, `retriever_index_hash`, `decoding_config_hash`, `seed` in a `fixtures/target.yaml` the spec reads.

5. Pick two rubrics — one numeric (faithfulness, groundedness) and one categorical (safety_refusal). Seed them in the `rubrics` table via exercise-01's back-fill. Pin a judge model snapshot per rubric; record the `rubric_hash` and `judge_model_snapshot` in the spec.

## Requirements

Produce a PR against your working branch that adds:

1. **`spec/*`** — pydantic models for `EvalRunSpec`, `EvalResultRow`, `RunManifest`, `CostBreakdown`. Chapter 04's shapes verbatim. Every field required unless chapter 04 marked it optional (e.g., `session_id`). `spec_hash` is computed from the canonicalised spec with `run_id`, `triggered_at`, `idempotency_key` excluded (test enforces).
2. **`adapters/base.py`** — the `RunnerAdapter` protocol with the four methods (`plan`, `run`, `stream_results`, `cost_report`), the `supported_metric_families` set, and the `name` string. A base helper `fill_lineage_from_spec(row, spec)` populates all nine lineage keys on every emitted row so no adapter has to re-implement it.
3. **Custom adapter (`adapters/custom/*`)** — the reference. In-process; iterates `eval_cases`, calls the target endpoint, calls a `scorer.py` that invokes the judge model directly, emits `EvalResultRow` as rows produce. The adapter has no vendor dependency. Smallest adapter in the exercise; longest-lived in your codebase. **This is the test harness for the plane itself** — if the plane's contract fits the custom adapter cleanly, it fits any adapter.
4. **OSS adapter (`adapters/<oss>/*`)** — Promptfoo, DeepEval, or RAGAS. The adapter:
   - **Translates** the `EvalRunSpec` into the runner's native config (Promptfoo YAML, DeepEval `LLMTestCase` list, or RAGAS HF `Dataset`).
   - **Invokes** the runner (subprocess for Promptfoo; in-process for DeepEval and RAGAS).
   - **Parses** the runner's native output and emits `EvalResultRow` with every lineage key filled in from the spec and `runner_native_row_id` set to the runner-native URI (`pf://`, `deepeval://`, `ragas://`).
   - **Reports cost** via `cost_report` — token counts × `pricing_snapshot` lookup. If the runner does not expose token counts directly, the adapter wraps the judge call to tally them.
5. **Hosted adapter (`adapters/<hosted>/*`)** — Braintrust, W&B Weave, or Langfuse. The adapter:
   - **Syncs the eval set** — on `plan`, creates or updates the hosted platform's dataset keyed by `eval_set_hash` (Braintrust `Dataset`, Weave `weave.Dataset`, Langfuse `Dataset`). The dataset is tagged with the `eval_set_hash` for traceability.
   - **Runs the eval** — `braintrust.Eval(...)` / `weave.Evaluation(...).evaluate(model)` / Langfuse `langfuse.score(...)` wired against the sync'd dataset.
   - **Streams rows back** — polls or subscribes to the hosted result stream, emits `EvalResultRow` with `runner_native_row_id` as the vendor URI (`brt://experiments/<id>/rows/<row>`, `wandb://.../runs/<id>`, `lfs://.../scores/<id>`).
   - **Reports cost** — the hosted SDK's cost / token API; fall back to `judgement.cost_usd` summation if the vendor's experiment-level cost call is unavailable.
   - **Vendor-UI link** — the manifest's `artefact_uris.vendor_native` points at the experiment in the hosted UI, clickable from exercise-05's PR comment.

   The adapter never retries on its own (chapter 04's rule — retries are the plane's responsibility).
6. **`control/dispatch.py`** — the plane's dispatcher. Signature `dispatch(spec: EvalRunSpec) -> RunManifest`. The dispatcher:
   - Resolves the `eval_set_hash` (if the spec names an id instead), freezes it on the spec, writes a row to `eval_runs` with `status=queued`.
   - Reads the cache per chapter 04's cache key `(rubric_hash, judge_model_snapshot, case_hash, target_output_hash)` before invoking the adapter; a hit emits the cached row with `cache_hit=true` and `cost_usd=0.0`.
   - Invokes the adapter's `plan` and `run` and streams `stream_results`.
   - On every emitted row: calls `control/normalise.py` (clamp score to the rubric's declared scale, canonicalise verdict strings), persists the row to `eval_results`, persists the `judgement` to `judgements`.
   - On termination: calls `cost_report`, updates `eval_runs.cost_usd`, `eval_runs.status`, writes the `run_manifest.json` to disk + the plane's HTTP artefact endpoint, emits the OpenLineage `RunEvent`.
7. **`control/normalise.py`** — the chapter 04 metric-normalisation contract. Reads the rubric spec from `rubrics.rubric_spec`; asserts the runner's native output conforms (score in declared scale, verdict in declared vocabulary); refuses the row with a specific error if not. The canonical metric name is the `rubric_id`; the runner's native name is a view only.
8. **`control/cache.py`** — a small persistent cache (SQLite is enough). Key: `sha256((rubric_hash, judge_model_snapshot, case_hash, target_output_hash))`. Value: the `EvalResultRow`. TTL 30 days default. Every adapter reads before invoking; the plane writes on every fresh scoring.
9. **CLI (`cli/plane.py`)** — `plane run <spec.json>` dispatches a spec and prints the manifest. `plane run --all-adapters <spec.json>` dispatches the same spec through every enabled adapter and prints a side-by-side manifest for the parity test.
10. **Headline test (`tests/test_three_runner_parity.py`)** — the proof the contract works. Runs the same `EvalRunSpec` (same `spec_hash`) through all three adapters and asserts:
    - Every adapter emits one row per (case × metric).
    - Every row has every lineage key populated (never `null` except `seed`).
    - For a *deterministic* subset (`seed != null`, judge temperature pinned), the normalised `score` values agree across adapters within a documented epsilon (typically `≤ 0.02` for a well-canonicalised numeric rubric — the slop comes from how each adapter rounds / clamps, not from the model).
    - The `runner_native_row_id` is a valid URI with the right scheme per runner.
    - Each adapter's `cost_report` sums to within `≤ 5 %` of a hand-computed `tokens × pricing_snapshot` reconciliation.
    - The three `run_manifest.json` files point at three different `vendor_native` URIs but at the same `openlineage_event` namespace.
11. **Adapter-boundary tests (`tests/test_adapter_contract.py`)** — exercises the chapter 04 "what the adapter does *not* do" list: an adapter that mutates `eval_runs` directly fails; an adapter that retries on a 429 fails (the plane does retries); an adapter that returns a row with a `null` `rubric_hash` fails; an adapter that adds a column to `eval_results` fails. These are *protocol* tests — they do not depend on the specific runner.
12. **README (`README.md`)** — three sections:
    - **For an adapter author.** How to implement a new adapter in ~ 200 lines; the four-method contract; the `fill_lineage_from_spec` helper; the "do not" list.
    - **For a plane operator.** How to run `plane run`; how to read the manifest; what the cache does and when to invalidate it; how to switch process models to out-of-process when the exercise outgrows in-process.
    - **For a downstream reader.** How to query `eval_results` by `run_id`, by `spec_hash`, by `eval_set_hash`, by `cost_centre`. The common joins.

## Starter guidance

- **Write the custom adapter first.** It is the simplest and it is the test harness for the plane. If the plane's contract does not fit a 100-line in-process Python evaluator, it will not fit a hosted vendor's SDK.
- **Pin the judge temperature.** The three-runner parity test is noisy if the judge sampling is stochastic. For the demonstration, set the judge decoding config to `temperature=0.0` and `seed=42` (where supported). Document the pin.
- **Keep the OSS adapter's subprocess invocation defensive.** Promptfoo and DeepEval sometimes exit non-zero on partial-success runs. Capture stdout + stderr, parse the structured output, let the plane handle retry classification; do not let the adapter decide what counts as "failed."
- **Do the hosted adapter's eval-set sync carefully.** Braintrust, Weave, and Langfuse each handle dataset upserts differently — some require a new dataset per `eval_set_hash`, some let you append. Read the vendor docs *at the time* you build the adapter (the SDKs change) and record which strategy your adapter chose in a comment, not a wiki page.
- **The `runner_native_row_id` URI is a contract, not a convenience.** A row without a valid native URI is a row that cannot be clicked through to the vendor's UI; the "mod-109 human-review workflow" (another module) depends on the click-through. Treat the URI as required.
- **Cache before the first judge call, not after.** The cache's whole value is that the second run against the same cases costs zero. Chapter 05 will show the cache hit rate as a cost-SLI tile.
- **Metric normalisation is where the slop lives.** Promptfoo's `llm-rubric` returns a 0 – 1 float; DeepEval returns a dict with `{score, reason, threshold}`; a Weave `Scorer` returns whatever the function returned. Document each adapter's translation into the canonical row in the adapter's module docstring.
- **Resist the urge to add a fifth method.** The four methods cover plan / dispatch / stream / cost. "Cancel" sounds necessary but is the plane's job (the dispatcher's `asyncio.CancelledError` propagates). "Validate" is the plan step. Protect the four-method contract.
- **In-process is fine for the exercise.** Out-of-process (chapter 04) is the stretch; the exercise's acceptance criteria read against the in-process shape. The contract does not change between the two.

## Acceptance criteria

You are done when:

- The three adapters each implement the four-method `RunnerAdapter` protocol; the contract tests pass against each.
- `plane run --all-adapters <spec.json>` dispatches the same spec through all three adapters and produces three manifests, each pointing at the correct runner-native URI.
- The parity test (`test_three_runner_parity.py`) passes — every row has every lineage key populated, the numeric score agrees within the documented epsilon on the deterministic subset, and the cost reconciliations agree within 5 %.
- The `eval_results` table has three sets of rows for the same `(eval_set_hash, rubric_hash)` combination — one per adapter — queryable with a single SQL across all three.
- The cache cuts the cost of a second identical run to zero (verify by running the spec twice and asserting the plane's judge spend is zero on the second).
- Every adapter emits a `run_manifest.json` with `status`, `runner`, `cost_usd`, `cost_centre`, `results_count`, `cache_hit_count`, and the three `artefact_uris` (eval-results query, vendor-native link, OpenLineage event).
- The contract tests reject an adapter that violates the chapter 04 boundaries (mutates the schema, implements its own retry, swallows a lineage key).
- The exercise-01 `reproducibility_check` passes on at least one run from each adapter — the point of the plane is that the schema's reproducibility guarantee survives any runner choice.
- The `README.md` sections for adapter author / plane operator / downstream reader exist and pass a reader check with a colleague who has not read chapter 04.

## Stretch goals

- **Fourth adapter.** Wire a fourth runner (if you did Promptfoo, add RAGAS; if you did Braintrust, add Weave). The parity test still passes with four runners; adding a fifth is still just a copy-paste of the base template.
- **Out-of-process dispatch.** Flip `process_model: out_of_process` in `config.yaml`. Replace the in-process adapter invocation with a Redis / RabbitMQ / SQS queue; the plane becomes a control-plane API and the adapters live in worker containers. The spec / result / manifest contract does not change — chapter 04's whole point.
- **Streaming vs batch dispatch.** The custom adapter streams rows as they produce; the hosted adapter batches rows at run end. Measure the plane's time-to-first-row on a 500-case run and tune the streaming to beat the batch by at least 10×.
- **Vendor-UI bidirectional sync.** Pair with exercise-02's "case added in the vendor UI → proposed contribution" path. The hosted adapter's `plan` step opens a proposal in the exercise-02 API when it finds a dataset row the plane did not emit.
- **Metric-approximation warnings.** When a runner approximates a metric the rubric spec declared exactly (e.g., Promptfoo's `llm-rubric` approximates a 1 – 5 Likert that the rubric spec defines as exact), the adapter's `plan` step emits a warning the dispatcher propagates onto the manifest's `warnings`. The exercise-05 PR comment surfaces the warning.
- **Cost-attribution forensics.** For the hosted adapter, cross-check the plane's `cost_usd` against the vendor's invoice line item for the run's experiment; write a one-liner forensics script that highlights discrepancies > 5 %.
- **Runner-health probe.** A `plane probe <runner>` CLI that sends a one-case spec through each adapter end-to-end, measures the time, and emits a small health row the chapter 06 dashboard tile reads. Run on a 5-minute schedule.

## What this exercise does *not* cover

You are not wiring the quotas, retry policy, rate-limit envelope, or cost-centre quota enforcement (those are exercise-05); you are not building the platform SLIs that watch the adapters' health (exercise-04); you are not integrating with the mod-106 PR gate or the mod-107 online loop (exercise-05). You are shipping the control-plane contract and three working adapters — the surface every other exercise in this module dispatches through.
