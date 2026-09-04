# Why an Eval-Data-Platform Slice: The Substrate the Previous Nine Modules Assumed

## Motivation

Every prior module in this track pointed at a store, a runner, or a test set and moved on. Mod-102 said "the spans land in Phoenix / Langfuse / Weave / Braintrust." Mod-104 said "the rubric is versioned." Mod-106 said "the replay bundle is content-addressed." Mod-107 said "the scored-row store is a table in the same warehouse." Mod-108 said "the OWASP-mapped report is a query." No module built the *thing being pointed at*. This module does.

The absence has a cost. Three symptoms that show a team has skipped the platform slice:

- **The same eval runs twice with different results and nobody can say why.** A PR gate scored 0.87 on faithfulness at 14:02 UTC. The online loop scored the same production traces at 0.79 at 14:07 UTC. Both used "the same rubric." Nobody logged which *rubric snapshot*, which *judge-model snapshot*, which *retriever index build*, which *decoding config* — so a week later, when a product owner asks whether the deploy caused the drop, the answer is "we can't tell." That is a lineage-schema problem, not a rubric problem.
- **Every product team has their own YAML directory of fixtures and none of them agree on shape.** Product team A's `test_cases.yaml` uses `expected: string`; team B's uses `expected: {answer, sources}`; team C uses a CSV. The harness has three code paths, one per shape, and adding a fourth team means adding a fourth path. Product-team A cannot re-use team B's cases because there is no shared vocabulary. That is a test-case-management problem, not a harness problem.
- **Two teams both hit the vendor rate limit at the same 09:00 UTC deploy window and both get paged.** No queue, no priority, no per-team quota, no idempotency key on the retry — so the retry storm makes the outage worse. Cost accounting is "we look at the vendor invoice on the 5th of the month." Nobody can answer "how much did team X's evals cost this week." That is a platform-operability problem, not a vendor problem.

Each of the three symptoms is a *platform* failure disguised as an application failure. This module builds the platform layer that makes the disguise impossible.

The scope is deliberate: **a slice**, not a full eval-as-a-service platform. The AI Evaluation Engineer role owns the *product-eval* half — the eval program for one or a few application surfaces, with lineage clean enough to reproduce a finding, a test-case store product teams can safely edit, one plane over the runners the org has picked, quotas and cost accounting for the teams that use the plane, and SLOs the eval team can actually be on-call for. The *org-wide, cross-modality, multi-tenant, capacity-planned* eval-as-a-service surface is a peer deliverable — `model-evaluation-engineer` at level 30. Chapter 07 draws the line.

## Core concepts

### What "slice" means and what it does not mean

A **product-eval slice** is enough platform to make one team's (or one product line's) eval program operable at production quality. It is:

- A queryable schema for traces + eval results + lineage.
- A test-case management surface product teams can contribute to.
- One plane over the eval runners the org has already deployed.
- Rate-limit / retry / quota / cost machinery around the plane.
- SLOs on the platform's own health, dashboarded and on-called.

It is **not**:

- A multi-tenant EaaS platform with per-tenant isolation guarantees and a paying-customer SLA (peer, `model-evaluation-engineer`).
- Cross-modality by construction — the slice may be text-only, or text-plus-tool-call-trace, without failing at its remit. Multimodal / image / audio evaluator infrastructure is peer.
- The rubric authorship or the runbook authorship (that is mod-104 / mod-106 / mod-108).
- Capacity planning across the whole eval fleet (peer).
- A general-purpose data platform. It uses the org's data platform where possible; it does not compete with it.

Confusing "slice" with "platform" is the single most common way this module goes wrong. The AI Evaluation Engineer who builds "the whole eval platform" is out of scope, over budget, and stepping on the peer track. The one who builds the slice, publishes its interface, and defends the delegation boundary is doing the job.

### The five load-bearing surfaces

The chapters build one surface each. Every surface has one job; each is required for the next to be operable.

| # | Surface | Chapter | What it owns | Why it is load-bearing |
|---|---|---|---|---|
| 1 | Lineage schema | 02 | The table shapes + lineage keys — `model_snapshot`, `prompt_hash`, `chain_hash`, `retriever_index_hash`, `decoding_config_hash`, `seed`, `eval_set_hash`, `rubric_hash`, `judge_model_snapshot` | Without lineage, a re-score cannot reproduce a result and a regression cannot be attributed. |
| 2 | Test-case management | 03 | Cases, suites, tags, ownership, versioning, contribution workflow | Without it, product teams cannot add cases without touching harness code; the eval set drifts silently. |
| 3 | Runner orchestration | 04 | The `EvalRunSpec` / `EvalResultRow` / runner-adapter contract | Without it, each vendor has its own SDK and results do not normalise across runs. |
| 4 | CI + online integration + quotas | 05 | The queue, rate-limits, retries, idempotency, per-team cost accounting, quota enforcement | Without it, two teams' deploys collide, retry storms compound outages, and cost is untraceable. |
| 5 | Platform SLOs | 06 | Success-rate SLO, latency-by-class SLO, cost-per-team drift SLO, queue-drop SLO, error budgets, on-call runbooks | Without them, the platform is a script — and a script that a product team depends on is a script that fails silently. |

Chapter 07 is the delegation boundary — it is *not* a sixth surface; it defines where the surfaces above end and where `model-evaluation-engineer` starts.

### The plane analogy

The word **plane** is borrowed from infrastructure. In a Kubernetes cluster, the control plane orchestrates workloads it does not itself perform; the data plane is where the workloads run. In a service mesh, the control plane configures proxies it does not proxy through; the data plane is the proxy fleet.

For eval, the same split:

- The **eval control plane** — this module — orchestrates runs it does not itself perform. It accepts an `EvalRunSpec`, dispatches it to a runner adapter, records the run in the lineage schema, tracks cost against a quota, emits results back into the eval-result store, and answers queries about them.
- The **eval data plane** — Promptfoo processes, DeepEval processes, RAGAS scoring functions, Braintrust project runs, Weave `weave.op` scorer calls, Langfuse evaluators, custom Python — actually loads test cases, calls models, runs judges, produces per-case results.

The plane does not re-implement any of the runners. It publishes one *contract* (`EvalRunSpec` in, `EvalResultRow` out, a `run_manifest.json`) and every runner is wrapped behind an adapter that speaks the contract. The plane does the *cross-cutting* work — lineage, quotas, retries, cost accounting, SLOs — that no single runner does and that every team otherwise re-invents.

The analogy also fixes what the plane is *not*: it is not a runner. If the team's temptation is "let's write our own evaluator that is better than Promptfoo," they have confused a control-plane responsibility with a data-plane responsibility. The plane wraps runners; it does not compete with them.

### The delegation boundary in one picture

```
   +-------------------------------------------+
   |         product-eval slice (this module)  |
   |                                           |
   |  lineage schema     test-case management  |
   |  runner-adapter plane  CI + online + quotas|
   |         platform SLOs                     |
   +---------------------+---------------------+
                         |
                         |   escalation contract (chapter 07)
                         v
   +-------------------------------------------+
   |   model-evaluation-engineer (peer, L30)   |
   |                                           |
   |  cross-modality evaluators                |
   |  multi-tenant isolation                   |
   |  paying-customer SLA                      |
   |  cross-product-line capacity planning     |
   |  eval-as-a-service surface                |
   +-------------------------------------------+
```

The interface between them is small and stable: the schema (chapter 02), the runner-adapter contract (chapter 04), the SLI history (chapter 06), and the test-case archive (chapter 03). If the peer has to re-derive any of those to absorb the slice, the boundary has been drawn wrong.

### Vocabulary the module reuses

Six terms recur through the chapters. Use them consistently in the repo — the runners and the vendors each call them different things, and drift between labels is how a schema ends up with three columns that all mean "run id."

- **Eval run.** One invocation of a runner against a specific `EvalRunSpec` — a triggered PR gate, a scheduled nightly, an online-loop scoring window. Every run has a `run_id`, a `spec`, a full lineage-key set, a start / end time, a status, a cost roll-up, and zero or more `EvalResultRow`s. The unit of billing, quota, and SLO.
- **Eval set.** A named, versioned collection of test cases (with a canonical content hash — the `eval_set_hash`). "The golden set for support-bot v3" or "the regression set for retrieval v11." Chapter 03 defines the contribution / tagging / versioning shape.
- **Lineage key set.** The fixed set of hashes and snapshots that together identify *exactly what was scored* — `model_snapshot`, `prompt_hash`, `chain_hash`, `retriever_index_hash`, `decoding_config_hash`, `seed`, `eval_set_hash`, `rubric_hash`, `judge_model_snapshot`. Chapter 02 walks each key; chapter 04 asserts that every runner adapter must populate them.
- **Runner adapter.** A small module implementing the plane's contract (`plan(spec) → PlannedRun`, `run(plan) → RunHandle`, `stream_results(handle) → iter[EvalResultRow]`, `cost_report(handle) → CostBreakdown`) on top of one specific runner (Promptfoo, DeepEval, RAGAS, Braintrust, Weave, Langfuse, custom). Chapter 04 fixes the contract; the exercise wires ≥ 3.
- **Cost centre.** The billing bucket a run charges against — usually `team_id` plus `project_id`. The quota, the alert threshold, the invoice roll-up, and the mod-107 online-loop budget are all indexed by cost centre. Chapter 05 covers the accounting rules.
- **Platform SLO.** A customer-visible commitment about the platform itself — `eval_run_success_rate ≥ 99.5 % over 7d`, `p95 latency of a class-A run ≤ 5 min`, `queue-drop rate ≤ 0.1 %`, `cost-per-team drift alert on > 20 % w-o-w`. Chapter 06 walks the four SLIs, the error-budget shape, and the on-call runbooks.

### Where each prior module reads back

The slice is not a new store hovering next to the old ones. Every prior module's artefact becomes a shape in the schema.

| Prior module | Artefact | Where it lives in the slice |
|---|---|---|
| mod-102 (traces) | OTel-GenAI spans | `traces`, `spans` tables (or the vendor's native span store, queryable through the plane) |
| mod-104 (rubrics) | Rubric YAML + hash | `rubrics` table; every eval result carries the `rubric_hash` |
| mod-104 (judge tier) | Judge-model snapshot | `judge_model_snapshot` column on every `judgement` row |
| mod-105 (RAG eval) | Retrieval-index build | `retriever_index_hash` in the lineage key set |
| mod-106 (offline gate) | Replay bundle + gate result | `eval_sets` (the replay), `eval_runs` (the gate), `gate_results` view |
| mod-107 (online loop) | Scored rows | `eval_results` rows with a `source=online_loop` tag |
| mod-108 (safety) | Findings + framework tags | A query over `eval_results` with `suite=safety_*` + a `findings` view |

Nothing is invented; everything is queryable together. That is the single deliverable that makes the eval program a *program* rather than a folder of scripts.

### What "operable" means for a platform

A product team depends on the plane. The moment they do, the plane has become a production system — the same rules mod-106 chapter 03 applied to the runbook apply to the plane itself:

- **Every alert links to a runbook that names an owner and a first step.** Not "the eval team looks into it." A named on-call rotation.
- **Every SLO has an error budget the team can spend or preserve deliberately.** Not "we aim for four nines and hope."
- **Every quota decision is auditable.** When team X's run is throttled or dropped, the log row names the quota, the current usage, and who owns the raise-quota decision.
- **Every cost line item is attributable to a cost centre.** Not "the vendor invoice was higher last month."
- **Every schema migration is versioned and reversible.** The lineage store has a longer half-life than any single runner or any single vendor account; treat it that way.

Chapter 06 walks this in detail. The exercise's platform-SLO deliverable is what turns *the script we wrote* into *the platform we operate*.

### Anti-patterns to avoid up front

Three failure modes worth naming before the chapters walk the mechanism:

- **The "vendor is the platform" anti-pattern.** Pick Braintrust (or Langfuse, or Weave) and declare that *is* the eval-data platform. The vendor stores runs; the vendor computes metrics; the vendor is the UI. What breaks: (a) the second runner has no home — Promptfoo runs go into a folder, DeepEval runs go into another folder, the vendor sees neither; (b) the lineage keys are whatever the vendor's data model happens to expose, not the set the reproducibility check needs; (c) cost accounting is per-vendor, not per-team. The vendor is a *data plane component*; the plane wraps it. Chapter 04 makes the wrap explicit.
- **The "one giant table" anti-pattern.** Score every result into a single 40-column wide table; add a JSON blob for the parts the schema does not accommodate. What breaks: the reproducibility check needs specific columns queryable; the platform SLO needs run-level state, not result-level rows; the test-case archive needs a stable case identity across suite versions. Chapter 02 walks the specific tables and their keys — the shape is boring but load-bearing.
- **The "build our own runner" anti-pattern.** The team's Promptfoo experience is bumpy; they conclude Promptfoo is bad; they write a new runner from scratch. Six months in, the new runner supports one metric family, one vendor, and one team's preferences. What breaks: the plane is a *control plane*, not a *runner*; the correct fix is a better adapter or an internal runner wrapped in an adapter, not "replace the industry with our own." Chapter 04 keeps the control-plane / data-plane split explicit.

### The end-of-module posture

By the end of this module you own the product-eval slice. Concretely, you can:

- Point at a lineage-schema table set and defend every column against a reproducibility test.
- Show a test-case management surface that a non-engineer product-team member has used to add a case, tag it, and see it in a run's results.
- Point at three runner adapters — hosted, OSS, custom — that all speak the plane's contract and normalise their results into the same eval-result table.
- Point at the mod-106 PR gate and the mod-107 online loop, and show every call is queued, rate-limited, retried with idempotency, and billed to a cost centre against a quota.
- Point at a platform dashboard with four SLIs, name the error budget for each, and read the current alert (or the last one) to a stranger.
- Hand the peer `model-evaluation-engineer` an escalation ticket that names the schema, the adapter interface, the SLI history, and the test-case archive as the inputs — and be able to defend where the slice ends.

The chapters build these one at a time. Chapter 02 starts with the schema — every downstream chapter reads it, and every prior module's artefact resolves to it.

## Summary

- The module builds a **product-eval slice** — enough platform to make one product line's eval program operable at production quality. It is *not* a cross-modality, multi-tenant eval-as-a-service platform (that is `model-evaluation-engineer`, peer, level 30).
- Five load-bearing surfaces: **lineage schema** (ch. 02), **test-case management** (ch. 03), **runner-adapter plane** (ch. 04), **CI + online integration + quotas** (ch. 05), **platform SLOs** (ch. 06). Chapter 07 defines the delegation boundary.
- The **plane analogy** separates a *control plane* (this module — lineage, quotas, retries, cost, SLOs) from a *data plane* (the runners — Promptfoo / DeepEval / RAGAS / Braintrust / Weave / Langfuse / custom).
- Six vocabulary terms recur — **eval run**, **eval set**, **lineage key set**, **runner adapter**, **cost centre**, **platform SLO**.
- Every prior module's artefact resolves into the schema; nothing is re-invented. The slice makes the whole eval program queryable together.
- **Operability** means the plane is on-call, alert-runbooked, error-budgeted, quota-audited, and cost-attributed — the same rules the mod-106 runbook applied to the app apply to the plane itself.
- The three failure modes to avoid up front: **"vendor is the platform"** (the vendor is a component, not the plane); **"one giant table"** (the schema shape matters); **"build our own runner"** (the plane wraps, it does not compete).

Chapter 02 opens the schema — the tables, the lineage keys, and the reproducibility contract every downstream chapter reads.
