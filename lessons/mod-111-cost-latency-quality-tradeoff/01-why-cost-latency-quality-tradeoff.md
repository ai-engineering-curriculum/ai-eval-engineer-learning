# Why Cost / Latency / Quality Trade-off Evaluation: The Decision Everything Else Compounds Into

## Motivation

Every prior module in this track produced a score. Mod-104 produced a rubric score for "was this answer faithful." Mod-105 produced a retrieval score for "did the retriever find the right document." Mod-106 produced a gate result for "does this PR pass." Mod-107 produced an online drift signal. Mod-108 produced a safety report. Mod-110 built the store that holds all of them.

None of that is a shipping decision.

A shipping decision is: *"we are going to serve traffic with model B instead of model A because on the eval set that matters, quality dropped by 1.2 rubric points on the tail cohort while cost dropped 62 % and streaming p95 latency dropped 380 ms, and product accepts that trade against the pre-registered rule."* Every clause in that sentence is load-bearing. Drop the cohort qualifier and you ship a model that regresses on the users who matter most; drop the cost or latency number and product cannot defend the change to finance or the on-call rotation; drop the pre-registered rule and every trade becomes a re-negotiation.

This module builds the report that carries all four numbers, the cohort-preservation contract that stops the tail from being averaged away, and the delegation contract that keeps the eval team from being asked to own model-altitude benchmarks or fleet-altitude kernels — which are peer jobs.

Three symptoms that show a team has skipped this module:

- **The org picks the model by asking each vendor for a demo and eyeballing the outputs.** A month later cost is 4× what was budgeted, one of the cohorts (say, non-English support) has silently regressed, and the on-call rotation is paging on TTFT the demo never surfaced. There was no report to defend or reject against.
- **A cheaper model is swapped in with a note "we A/B'd it and quality looks fine."** No paired eval set, no cohort breakout, no pre-registered decision rule. Two months later a specific product surface has degraded — but the swap-in cannot be attributed because the online rubric averaged across all surfaces.
- **The monthly vendor invoice arrives 3× the prior month and no one can tell which feature drove the growth.** Finance opens a ticket asking for a cost breakdown per feature; the eval team says "we have per-model token counts from the vendor console"; finance says "that is not what I asked for." The token accounting is per-vendor, not per-feature.

Each symptom is a *decision-surface* failure disguised as a modelling, product, or accounting failure. This module builds the decision surface so the disguise is impossible.

The scope is deliberate: **the app-altitude trade-off surface**, not the model-altitude benchmark suite or the fleet-altitude serving stack. The AI Evaluation Engineer owns quality-as-users-experience-it, cost-as-the-invoice-lines-say, latency-as-the-p95-streaming-metric. The *model-altitude* benchmark suite (MMLU, GPQA, HumanEval, MMMU, LiveCodeBench, whatever the org standardises on) is a peer deliverable — `model-evaluation-engineer` at level 30. The *fleet-altitude* serving stack (kernels, batching, KV-cache management, GPU-hours-per-token) is `ai-infra-performance`. Chapter 07 draws the lines.

## Core concepts

### The four axes

Every shipping decision has to price at least these four axes. Skip any one and the trade is not defensible.

| # | Axis | What it measures | Where it comes from | Failure mode when skipped |
|---|---|---|---|---|
| 1 | **Quality** | The rubric scores against the eval set that matters — including per-cohort breakouts | Mod-104 rubrics, mod-105 RAG rubrics, mod-108 safety rubrics | Ship a regression because the average moved 0.01 but the tail cohort dropped 1.5 |
| 2 | **Cost** | Dollars per request, per feature, per team — with input / output / cache-read / cache-write / batch tokens split | Vendor pricing snapshots × token counts from the trace pipeline | Ship a model that is "cheaper per token" but 4× the tokens per request and 2× the cost per user |
| 3 | **Latency** | TTFT, TPOT, streaming p50 / p95, retry latency — measured at the app altitude the user sees | OpenTelemetry GenAI spans from mod-102 | Ship a model whose model-altitude latency is fine but whose app-altitude p95 is 3× worse because of a retry loop and tool-call round-trips |
| 4 | **Safety** | The OWASP LLM Top-10 mapped runbook from mod-108 | Mod-108 safety report + on-call findings | Ship a model with better quality-and-cost but worse jailbreak resistance; find out from the abuse team the next week |

Only after all four are on the report can product ask the question the report was built to answer: **"given these four numbers on these cohorts, do we ship?"** That question has a defensible answer. "Which model has the highest MMLU score?" does not.

### The plane analogy: app-altitude, model-altitude, fleet-altitude

Borrowed from the aviation / infrastructure vocabulary the prior modules use. Three altitudes; each has its own eval surface; each owned by a different role.

```
   +-----------------------------------------------------+
   |   app altitude — THIS MODULE                       |
   |   (AI Evaluation Engineer)                          |
   |                                                     |
   |   quality-on-the-cohorts-that-matter                |
   |   cost-per-feature-and-per-team                     |
   |   TTFT / TPOT / streaming-p95 the user sees         |
   |   OWASP LLM Top-10 posture in production            |
   +----------------+------------------------------------+
                    |
                    | reads down; publishes back up
                    v
   +-----------------------------------------------------+
   |   model altitude — model-evaluation-engineer peer   |
   |   (Model-Development family, L30)                    |
   |                                                     |
   |   MMLU, GPQA, HumanEval, MMMU, LiveCodeBench, ...   |
   |   pass@1 on offline suites                          |
   |   MLPerf Inference — Server / Offline scenarios     |
   |   model-family capability profiles                  |
   +----------------+------------------------------------+
                    |
                    | reads down; publishes back up
                    v
   +-----------------------------------------------------+
   |   fleet altitude — ai-infra-performance peer        |
   |                                                     |
   |   kernels, batching, KV-cache, scheduler            |
   |   GPU-hours per million tokens                      |
   |   node-level utilisation, memory bandwidth          |
   |   capacity planning across the serving cluster      |
   +-----------------------------------------------------+
```

- **App altitude — this module.** What the user experiences. The eval sets, the rubrics, the cohorts, the streaming latency, the per-feature spend. The trade-off report lives here.
- **Model altitude — `model-evaluation-engineer`.** What the model, in isolation, is capable of. Offline benchmark suites, capability profiles, MLPerf-shaped serving benchmarks (where "serving" still means "one model, one dataset, one hardware target"). The report reads *down* into this — a mid-tier candidate's model-altitude benchmark tells you whether it is even plausible to try — and publishes back *up* — an app-altitude regression on a cohort feeds into the peer's next capability profile.
- **Fleet altitude — `ai-infra-performance`.** What the serving stack does with the model on the org's hardware. Kernel-level throughput, KV-cache eviction policy, batch shape, GPU utilisation, capacity plan. The report reads *down* — fleet-level tail-latency changes flow up as a lineage key on the app-altitude latency report — and publishes back *up* — a sustained p95 regression the app-altitude report finds is a fleet-altitude backlog item.

The analogy fixes what this module is *not*. It is not the peer that runs MMLU every model-family bump. It is not the peer that owns the batching config. It is the app-altitude surface that reads both peers, adds the user-visible / cohort-preserving / per-feature-attributed dimensions the peers do not, and produces the shipping decision.

### The six surfaces

The chapters build one surface each. Every surface has one job; each is required for the next report to be defensible.

| # | Surface | Chapter | What it owns | Why it is load-bearing |
|---|---|---|---|---|
| 1 | Multi-objective report | 02 | The report shape — four columns, cohort breakouts, Pareto view, pre-registered decision rule | Without it, decisions default to whichever axis is loudest in the room. |
| 2 | Model routing eval | 03 | The cohort-preservation contract, the routing report, the router-error separation | Without it, a router regresses the tail cohort while looking fine on the average. |
| 3 | Replacement regression | 04 | The paired-eval methodology, the delta report, the ship / hold decision matrix | Without it, a swap-in is a launch — no attribution, no rollback trigger. |
| 4 | App-altitude latency report | 05 | TTFT, TPOT, streaming p50 / p95, retry latency; reconciliation with MLPerf | Without it, the latency number on the report is a vendor claim, not a user experience. |
| 5 | Per-feature token accounting | 06 | The cost report per feature and per team; the pricing snapshot; cache and batch accounting | Without it, cost is a vendor invoice, not an eval artefact that drives product decisions. |
| 6 | Delegation to peers | 07 | The escalation shape to `model-evaluation-engineer` and `ai-infra-performance` | Without it, the eval team ends up owning benchmarks or kernels — jobs it is not staffed for. |

Chapter 07 is the delegation boundary — it is *not* a sixth report; it defines where the app-altitude surface ends and where the peers pick up.

### Vocabulary the module reuses

Six terms recur through the chapters. Use them consistently in the repo — the vendors, the papers, and the runners each label them differently, and drift between labels is how a trade-off report ends up with three columns that all mean "cost."

- **Decision candidate.** A specific configuration the report evaluates end-to-end: a model snapshot, a prompt, a chain, a retriever, a decoding config, an optional router policy, an optional cache policy. Never "the model" alone — the trade-off attaches to the *candidate*, not to any single component. Every candidate has a stable `candidate_id` and a lineage-key set (chapter 02 of mod-110).
- **Pareto frontier.** The subset of candidates that no other candidate dominates on all four axes. A candidate on the frontier is a legitimate shipping option even if it does not win on any single axis; a candidate off the frontier is dominated — pick the dominator. Chapter 02 defines the shape.
- **Cohort-preservation contract.** A pre-registered guarantee that no named cohort's quality drops by more than δ (a small, published number — often 0.5 – 1.0 rubric points on a 0 – 5 scale, or 2 – 3 percentage points on a pass-rate) versus a named baseline. Chapter 03 formalises this for routing; chapter 04 formalises it for replacement.
- **Replacement regression.** A paired-comparison experiment on a frozen eval set where the same inputs are run through the incumbent and the candidate; per-case deltas are what the report shows, not per-suite averages. Chapter 04 defines the shape.
- **App-altitude latency.** The latency the user perceives, measured at the trace-root span the mod-102 instrumentation emits: TTFT (time to first token), TPOT (time per output token), streaming p50 / p95, retry latency, tool-call round-trip. Distinct from *model-altitude latency* (single-shot forward pass on the vendor's benchmark rig — MLPerf-shaped). Chapter 05 walks each.
- **Per-feature cost centre.** The billing bucket a request charges against — usually `product_line / feature / cohort` — populated as a span attribute at the app boundary. The invoice roll-up, the per-feature ship / hold decisions, and the mod-110 quota-per-team all index by this. Chapter 06 covers the accounting rules.

### The report is a decision artefact, not a dashboard

The single most common failure of this work is producing a dashboard. Dashboards are read; reports are *signed*.

A dashboard says "here are the numbers, updated every 5 minutes." A report says "here are the numbers, frozen at commit `abc123`, evaluated against the pre-registered rule `X`, and the decision is `ship / hold / roll back`." The report has a named owner, a named signer, an artefact hash, and a decision. The dashboard is a debugging tool the report reader might drill into after the fact.

Concretely:

- The report has a **timestamp and a lineage-key set** (mod-110 chapter 02). Reading the same report tomorrow returns the same numbers.
- The report has a **pre-registered decision rule** (chapter 02) — the rule was written before the numbers were seen. "Ship if quality on cohort `enterprise-support` does not drop more than 0.5 rubric points AND cost drops at least 30 %" is a rule. "It looks good, let's ship" is not.
- The report has a **signer** — a named human whose sign-off is required for the decision to become an action. The eval team produces; product signs.
- The report has a **rollback trigger** — a condition against the online loop's metrics (mod-107) under which the decision reverses automatically.

Chapter 02 walks the report shape; chapters 03 – 06 add the four surfaces the report needs to be defensible.

### Where each prior module reads back

The report is not a new store hovering next to the old ones. Every prior module's artefact becomes a column or a filter on the report.

| Prior module | Artefact | Where it appears on the trade-off report |
|---|---|---|
| mod-102 (traces) | OTel-GenAI spans | The latency columns (TTFT / TPOT / streaming p95); the per-feature attribution key on every request span |
| mod-104 (rubrics) | Rubric scores | The quality column and its per-cohort breakout |
| mod-105 (RAG eval) | Retrieval + faithfulness scores | Quality sub-columns for RAG surfaces |
| mod-106 (offline gate) | Replay bundle + gate result | The eval set the report is scored against |
| mod-107 (online loop) | Scored rows, drift signals | The online-loop rollback trigger; the "quality-as-observed-in-production" back-check |
| mod-108 (safety) | Findings + OWASP tags | The safety column |
| mod-110 (platform slice) | `eval_runs`, `eval_results`, `pricing_snapshots` | The store the report queries |

Nothing is invented; everything is queryable together against the mod-110 schema. The report is the query, not a new store.

### What "defensible" means for a trade-off report

Product asks the report reader four questions when the report is being defended. If any answer is "we don't know," the report is not yet defensible.

- **On which eval set?** Named, versioned, hash-pinned. If the eval set is "the one I ran last week," the report is not defensible. Mod-106's replay bundle and mod-110's `eval_set_hash` are the answer.
- **Against which baseline?** A named incumbent candidate with its own `candidate_id`. If the baseline is "the previous model," the report is not defensible — the previous prompt, chain, retriever, and decoding config are all part of the incumbent.
- **On which cohorts?** Named, populated at request time, present as a column on every result row. If the cohorts are "we sliced by locale after the run," the report is not yet defensible — the cohort has to be pre-declared or the cohort-specific numbers are cherry-picked.
- **Against which pre-registered decision rule?** Written before the numbers were seen, signed off by product. If the rule is "let's see the numbers and decide," the report is a dashboard, not a report.

Chapters 02 – 06 walk each of these. The exercises make each answerable in code.

### Anti-patterns to avoid up front

Four failure modes worth naming before the chapters walk the mechanism:

- **The "single-number score" anti-pattern.** Collapse the four axes into a weighted sum (`0.5 * quality + 0.2 * -log(cost) + 0.2 * -log(latency) + 0.1 * safety`) and pick the highest. What breaks: the weights are un-defendable ("why 0.5 not 0.6?"), a candidate that is safety-average and cost-great wins over a safety-great cost-average candidate that the abuse team would prefer, and the Pareto view is destroyed by the collapse. Chapter 02 keeps the axes separate.
- **The "average quality" anti-pattern.** Report `mean(quality)` across the eval set. What breaks: the tail cohort — enterprise / regulated / non-English / long-context — is one of five, so a 2-point regression there moves the mean by 0.4 and looks like "noise." Chapter 03 formalises the cohort-preservation contract; chapter 04 requires per-cohort deltas on the replacement report.
- **The "vendor latency" anti-pattern.** Take the p50 the vendor console shows and call it the app latency. What breaks: the vendor's number is the model-altitude latency for a single request; the app's number includes prompt-build time, tool-call round-trips, retries, streaming buffering, and the multi-model chain. Chapter 05 walks the app-altitude measurement.
- **The "vendor invoice" anti-pattern.** Wait for the vendor invoice, then attribute after the fact. What breaks: the invoice is per-model, not per-feature; the retro-attribution is heuristic ("this project uses this model, and their usage grew 3× so their cost grew 3×"); the report cannot drive product decisions because the growth cause is unknown. Chapter 06 walks the pre-attribution.

### The end-of-module posture

By the end of this module you own the app-altitude trade-off surface. Concretely, you can:

- Show a multi-objective report — four columns, per-cohort breakout, Pareto view, pre-registered decision rule — for at least one real shipping decision the product owner signed against.
- Show a model-routing eval that names the cohort-preservation contract, the router-error separation, and the per-cohort quality delta against the frontier baseline.
- Show a replacement-regression report that names the paired eval set, the per-cohort deltas, the ship / hold decision against the pre-registered rule, and the rollback trigger.
- Show an app-altitude latency report — TTFT, TPOT, streaming p50 / p95, retry latency — reconciled against the model-altitude benchmark (MLPerf-shaped, if the org has it) with the discrepancy attributed to a specific app-layer cause.
- Show a per-feature cost report — input / output / cache-read / cache-write / batch tokens, per feature, per team — that has driven at least one product decision (kill, downgrade, cache, batch).
- Hand the peers `model-evaluation-engineer` and `ai-infra-performance` a clean escalation ticket that names the app-altitude gap and asks the right question at the right altitude.

The chapters build these one at a time. Chapter 02 opens the multi-objective report — every downstream chapter's numbers land in it.

## Summary

- The module builds **the decision surface** the previous ten modules feed into — a multi-objective eval report that lets product decide what to ship when quality, cost, latency, and safety pull in different directions.
- The **four axes** are quality, cost, latency, safety. Skip any one and the trade is not defensible.
- The **plane analogy** — app altitude (this module) reads down from model altitude (`model-evaluation-engineer`, L30) and fleet altitude (`ai-infra-performance`), and publishes back up. This module does not own model-altitude benchmarks or fleet-altitude kernels.
- Six load-bearing surfaces: **multi-objective report** (ch. 02), **model routing eval** (ch. 03), **replacement regression** (ch. 04), **app-altitude latency report** (ch. 05), **per-feature token accounting** (ch. 06). Chapter 07 is the delegation boundary.
- Six vocabulary terms recur — **decision candidate**, **Pareto frontier**, **cohort-preservation contract**, **replacement regression**, **app-altitude latency**, **per-feature cost centre**.
- The report is a **decision artefact**, not a dashboard — timestamped, lineage-pinned, pre-registered decision rule, signed by product, with a rollback trigger.
- Every prior module's artefact resolves into a column or filter on the report; nothing is re-invented. The report is a query over the mod-110 schema.
- Four anti-patterns to avoid up front: **"single-number score"** (destroys the Pareto view), **"average quality"** (hides the tail cohort), **"vendor latency"** (ignores app-layer overhead), **"vendor invoice"** (attribution after the fact cannot drive decisions).

Chapter 02 opens the multi-objective report — the shape every downstream chapter's numbers land in.
