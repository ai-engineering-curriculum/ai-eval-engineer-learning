# mod-110 — Eval-Data-Platform Slice for AI Applications

**Estimated effort:** 16 hours

This is the tenth module of the AI Evaluation Engineer track. It builds the *platform slice* the previous nine modules have been silently assuming: a trace + eval-result warehouse with real lineage, a test-case management layer product teams can safely contribute to, a single control plane over the eval runners the org has already picked (Promptfoo, DeepEval, RAGAS, Braintrust, W&B Weave, Langfuse, and whatever custom evaluators the team wrote), a CI and online-eval integration with rate-limits, retries, and cost accounting per team, and SLOs on the platform itself so it stays operable under load.

Mod-102 built the trace instrumentation and pointed at a backend. Mod-104 defined the rubrics. Mod-106 wired the offline gate. Mod-107 built the online loop with its scored-row store. Mod-108 published the safety report. Every one of those modules touched an "eval store" or an "eval runner" or a "test set" and moved on — because *this* module is where the substrate actually lives. Without it, each product team ends up with their own YAML directory, their own vendor account, their own CSV of fixtures, and no one can compute a cross-team eval-run success-rate SLO because there is no cross-team ledger to compute it against.

The module deliberately does not build a **cross-modality, multi-tenant eval-as-a-service platform** (that is a `model-evaluation-engineer` deliverable — the peer track at level 30 owns the depth). It builds the **product-eval slice** — enough platform to make the AI Evaluation Engineer's own program operable and to give the peer a clean interface when the org scales up to a real eval platform.

## Learning objectives

- Design a trace warehouse + eval-result store schema with traceable lineage: model / prompt / chain / retriever / decoding config / seed / eval-set hash.
- Design a test-case management layer that lets product teams contribute, tag, and version test cases without touching the harness code.
- Orchestrate multiple eval runners behind one plane: Promptfoo, DeepEval, RAGAS, Braintrust, W&B Weave, Langfuse, and custom evaluators.
- Wire the eval platform into CI and into the online-eval loop with rate-limits, retries, and cost accounting per team.
- Instrument the eval platform itself with SLO / SLA (eval-run latency, success rate, cost per team) so it is operable — not just a script.
- Coordinate with `model-evaluation-engineer` when the org needs cross-modality / multi-tenant eval-as-a-service depth beyond a product-eval slice.

## Lecture chapters

1. [Why an eval-data-platform slice](01-why-eval-data-platform-slice.md) — what "slice" means; the five load-bearing surfaces (lineage schema, test-case management, runner orchestration, CI + online integration, platform SLOs); the plane analogy; the delegation boundary to `model-evaluation-engineer`; six vocabulary terms the module reuses (**eval run**, **eval set**, **lineage key set**, **runner adapter**, **cost centre**, **platform SLO**).
2. [Trace warehouse and eval-result store schema with lineage](02-trace-warehouse-and-eval-result-schema.md) — the load-bearing tables (`traces`, `spans`, `eval_sets`, `eval_cases`, `eval_runs`, `eval_results`, `judgements`, `rubrics`); the lineage key set (`model_snapshot`, `prompt_hash`, `chain_hash`, `retriever_index_hash`, `decoding_config_hash`, `seed`, `eval_set_hash`, `rubric_hash`, `judge_model_snapshot`); why the hashes are content-hashes not version tags; storage shape (row store for hot, columnar for warm); OpenLineage as the interchange format; the reproducibility contract.
3. [Test-case management and versioning](03-test-case-management-and-versioning.md) — the "product teams contribute cases" problem; the case-id contract; the eval-set-as-code pattern (YAML in repo, PR review, CI validation) vs the eval-set-as-database pattern (UI plus API, four-eyes review); tags, ownership, retention; semver on the suite; the `eval_set_hash` that ties every result back to the exact contents scored; the golden / regression / rot-check subset structure.
4. [Multi-runner orchestration behind one plane](04-multi-runner-orchestration-behind-one-plane.md) — why an org has more than one runner (per-team preference, per-eval-shape suitability); the runner-adapter interface (`plan`, `run`, `stream_results`, `cost_report`); adapters for Promptfoo, DeepEval, RAGAS, Braintrust, W&B Weave, Langfuse, and custom evaluators; the plane's contract with each; where the metric normalisation happens; the `EvalRunSpec` and `EvalResultRow` that flow across the boundary.
5. [CI and online-eval integration with quotas](05-ci-and-online-eval-integration.md) — the two integration surfaces (mod-106 offline gate at PR and deploy events; mod-107 online sampled-scoring loop); the queue design (per-team weighted-fair queue with priorities); rate-limits, retries with backoff, idempotency keys; cost accounting per team and per project (`cost_centre`, `pricing_snapshot`, `unit_cost`); quota enforcement (soft warn at 80 %, hard cap at 100 %, override token pattern); the artefacts the platform emits back to CI (`gate_result.json`, `run_manifest.json`).
6. [Platform SLOs and operability](06-platform-slos-and-operability.md) — the four platform SLIs the eval team is on-call for (eval-run success rate, eval-run latency by class, cost-per-team drift, queue-drop rate); the error budget shape; SLOs as customer-visible commitments not internal targets; the platform's own dashboard (green / yellow / red per SLI); alerts and runbooks the eval team owns; capacity planning against the mod-107 sample-rate math; where the platform is *not yet* the org's primary eval-serving surface (delegation).
7. [Delegation to model-evaluation-engineer for platform-scale depth](07-delegation-to-model-evaluation-engineer.md) — where the product-eval slice ends and cross-modality, multi-tenant eval-as-a-service begins; the peer's remit (image / audio / multimodal evaluators, per-tenant isolation, cost governance across product lines, capacity planning across the whole eval fleet, an SLA promise a paying internal customer relies on); the delegation contract (what this module hands over: the schema, the adapter interface, the SLI history, the test-case archive); how to write the escalation ticket; the difference between "we outgrew our slice" and "we mis-sized the slice."

## Exercises

Each exercise builds on the last. Do them in order — later exercises depend on the schema, the test-case store, the runner-adapter interface, and the quota table produced by earlier ones. The five deliverables together are a working *slice* — they are not a replacement for a full eval-as-a-service platform, and the module makes that difference explicit.

1. [Trace and eval-result schema with lineage](exercises/exercise-01-trace-and-eval-result-schema-with-lineage.md) — design and materialise the chapter 02 schema in a real store (Postgres + a columnar warm tier, or the vendor's native tables); wire the lineage-key hash computation; back-fill from your mod-102 trace store and mod-107 scored-row store; write the reproducibility check.
2. [Test-case management and versioning](exercises/exercise-02-test-case-management-versioning.md) — implement the eval-set-as-code contribution flow (YAML in repo, PR review, CI validation, semver bump), plus a small contribution UI or API that non-engineer product teams can use; wire ownership, tags, and the `eval_set_hash`.
3. [Multi-runner orchestration behind one plane](exercises/exercise-03-multi-runner-orchestration-behind-one-plane.md) — ship the plane and the runner-adapter interface; wire adapters for at least three runners (one hosted, one OSS, one custom); run the same `EvalRunSpec` through all three and confirm normalised results in the eval-result store.
4. [Platform SLOs and cost per team](exercises/exercise-04-platform-slos-and-cost-per-team.md) — instrument the four SLIs; publish SLOs and error budgets; wire cost accounting per team and per project; ship the platform's own dashboard; write the runbook for the two most likely alerts.
5. [CI and online-eval integration](exercises/exercise-05-ci-and-online-eval-integration.md) — wire the platform into the mod-106 PR gate and the mod-107 online loop with rate-limits, retries, idempotency keys, per-team quotas, and the two artefacts (`gate_result.json`, `run_manifest.json`) CI reads.

## Labs and quizzes

- Labs (see [`labs/`](labs)) build an end-to-end product-eval platform slice on top of the mod-102 trace backend, the mod-107 scored-row store, and the runners the org has already deployed: the schema with lineage, the test-case management surface, the runner-adapter plane, the CI + online integration with quotas, and the four platform SLIs. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary — the six terms, the load-bearing lineage keys, the eval-set-as-code / eval-set-as-database distinction, the runner-adapter contract, the two integration surfaces, the four platform SLIs, and the delegation boundary to `model-evaluation-engineer`. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- **Cost / latency / quality trade-off** reads the per-team cost accounting from chapter 05 and the platform-SLI history from chapter 06 to drive tenant-facing trade-off dashboards → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- **Program-owner posture** treats the schema, the test-case archive, the runner-adapter plane, and the platform-SLI dashboard as load-bearing program artefacts — the ones that survive a change of eval vendor, judge model, or product owner → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
- **Model-evaluation-engineer (peer track, level 30)** owns the org-wide, cross-modality, multi-tenant eval-as-a-service surface. Chapter 07 defines what this module hands over when the org outgrows a product-eval slice — the schema, the adapter interface, the SLI history, the test-case archive — and what it does *not* hand over (rubric authorship, application-specific runbooks, product ownership).
- **Every prior module** reads back cleaner: mod-104's rubric hashes land in the `rubrics` table; mod-106's replay bundle becomes an `eval_set` with a canonical hash; mod-107's scored rows are one shape of `eval_result` row; mod-108's safety report is a query over the same store with an OWASP tag. The slice does not replace the prior work — it makes it queryable.
