# mod-111 — Cost / Latency / Quality Trade-off Evaluation

**Estimated effort:** 12 hours

This is the eleventh module of the AI Evaluation Engineer track. It builds the *decision surface* the previous ten modules have been feeding into: a **multi-objective eval report** that lets product decide what to ship when the quality winner is not the cost winner, and the cost winner is not the latency winner, and every one of them may or may not be the safety winner.

Mod-104 defined the quality rubrics. Mod-105 defined the retrieval-quality rubrics. Mod-106 built the offline gate. Mod-107 built the online loop. Mod-108 published the safety report. Mod-110 built the platform slice that stores every one of those results side-by-side. This module reads that store and produces the trade-off report — Pareto-fronted, per-cohort-preserved, with per-feature token accounting — that product uses to pick a model, pick a route, pick a distillation swap, or pick a latency budget without flying blind on the other three axes.

The module deliberately does not build **model-altitude offline benchmarks** (MMLU / GPQA / HumanEval / whatever suite the org standardises on) or **fleet-altitude serving benchmarks** (MLPerf Inference, GPU-hours-per-token, kernel-level optimisation). Those are peer deliverables — [`model-evaluation-engineer`](https://github.com/anthropics/skills) owns model-altitude quality at level 30, and `ai-infra-performance` owns the serving fleet. This module builds the **app-altitude trade-off surface** — quality-as-users-experience-it, cost-as-the-invoice-lines-say, latency-as-the-p95-streaming-metric — and reconciles it with the peers' deeper benchmarks. Chapter 07 defines the delegation.

## Learning objectives

- Design a **multi-objective eval report** (quality vs. cost vs. latency vs. safety) that lets product decide what to ship.
- Evaluate **model routing** (frontier vs. mid-tier vs. open-source, or classifier-gated routing) with **per-cohort quality preserved**.
- Run **distillation / replacement regression tests**: swap a cheaper model in and prove where quality moves.
- Measure **app-altitude latency the way users experience it** (TTFT, TPOT, streaming p50 / p95, retry latency) and reconcile with the MLPerf-shaped serving benchmarks owned by `model-evaluation-engineer` and `ai-infra-performance`.
- Track **token accounting per feature** so an eval-driven cost report can drive product decisions.

## Lecture chapters

1. [Why cost / latency / quality trade-off evaluation](01-why-cost-latency-quality-tradeoff.md) — the decision that everything else compounds into; the four axes and why a single-axis winner is almost always the wrong answer; the plane analogy (app-altitude vs. model-altitude vs. fleet-altitude); six vocabulary terms the module reuses (**decision candidate**, **Pareto frontier**, **cohort-preservation contract**, **replacement regression**, **app-altitude latency**, **per-feature cost centre**); the delegation boundary to `model-evaluation-engineer` and `ai-infra-performance`.
2. [The multi-objective eval report](02-multi-objective-eval-report.md) — the four columns (quality, cost, latency, safety); the report as a decision artefact, not a dashboard; Pareto frontier and dominance; tie-breakers; the pre-registered decision rule; who signs off; the anti-patterns (single-number scores, missing safety column, cohort-averaged quality, latency-mean-only).
3. [Model routing evaluation with cohort preservation](03-model-routing-eval-with-cohort-preservation.md) — routing patterns (tier-based, classifier-gated, difficulty-scored, cache-first); the **cohort-preservation contract** (no cohort's quality drops more than δ vs. the frontier baseline); the routing report shape; measuring routing quality without leaking the router's own errors into the tier's numbers; the "always-frontier-when-unsure" fallback and how to eval it.
4. [Distillation and replacement regression](04-distillation-and-replacement-regression.md) — the swap-in as a regression event, not a launch; paired-comparison methodology on the same eval set; the **replacement regression** contract; distillation quality attribution (student vs. teacher gap); the delta report — quality moved by X on cohort Y, cost dropped by Z, latency dropped by W; the shipping decision matrix.
5. [App-altitude latency the way users experience it](05-app-altitude-latency-report.md) — TTFT, TPOT, streaming p50 / p95, retry latency, end-to-end vs. per-token; the app-altitude / model-altitude / fleet-altitude split; the OpenTelemetry GenAI attributes for latency; reconciliation with MLPerf Inference on the model-altitude side; the tail-latency budget as a first-class product decision.
6. [Per-feature token accounting](06-per-feature-token-accounting.md) — the cost report as an eval artefact; per-feature attribution keys; prompt caching (Anthropic prompt caching, OpenAI Prompts / prompt caching, Gemini context caching); Batch API accounting; input / output / cache-read / cache-write token classes; the pricing snapshot; how the accounting drives product decisions ("Feature X is 40 % of monthly spend for 3 % of value — kill or downgrade").
7. [Delegation to model-evaluation-engineer and ai-infra-performance](07-delegation-to-model-evaluation-engineer.md) — where app-altitude ends and model-altitude / fleet-altitude begin; the peers' remits (offline benchmarks, MLPerf-shaped serving, kernel-level optimisation, capacity planning); the delegation contract (what this module hands off, what it does not); the escalation shape; failure modes on both sides.

## Exercises

Each exercise builds one of the load-bearing artefacts. Do them in order — the multi-objective report shape from exercise 01 is the shape every subsequent exercise reports against. Exercise 04's app-altitude latency measurement is the substrate for exercise 05's cost-per-token-and-per-request accounting. The five deliverables together are the trade-off surface a product owner can defend a ship / hold / rollback decision on.

1. [Multi-objective eval report](exercises/exercise-01-multi-objective-eval-report.md) — design and ship the chapter 02 report against a real decision (e.g., pick between three candidate models for the support-bot); wire the four columns to your mod-110 store; publish the Pareto view and the pre-registered decision rule.
2. [Model routing eval with cohort preservation](exercises/exercise-02-model-routing-eval-with-cohort-preservation.md) — design a two- or three-tier routing policy; measure per-cohort quality preservation against the frontier baseline; publish the routing report and the cohort-preservation contract.
3. [Distillation and replacement regression](exercises/exercise-03-distillation-and-replacement-regression.md) — swap a cheaper model in behind a paired eval-set comparison; publish the delta report (quality moved by X on cohort Y, cost dropped by Z, latency dropped by W); make the ship / hold decision against the pre-registered rule.
4. [App-altitude latency report](exercises/exercise-04-app-altitude-latency-report.md) — instrument TTFT, TPOT, streaming p50 / p95, retry latency; publish the app-altitude latency report; reconcile against the MLPerf-shaped serving benchmark if the org has one.
5. [Per-feature token accounting](exercises/exercise-05-per-feature-token-accounting.md) — wire per-feature attribution into the trace pipeline; account for input / output / cache-read / cache-write and Batch API tokens; publish the per-feature cost report; drive at least one product decision from it.

## Labs and quizzes

- Labs (see [`labs/`](labs)) build the end-to-end trade-off surface on top of the mod-110 platform slice: the multi-objective report, the routing eval, the replacement-regression harness, the app-altitude latency report, and the per-feature cost report — with the pre-registered decision rule, the cohort-preservation contract, and the delegation escalation ticket. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary — the six terms, the four report columns, the cohort-preservation contract, the replacement-regression shape, the app-altitude / model-altitude / fleet-altitude split, and the delegation boundary to `model-evaluation-engineer` and `ai-infra-performance`. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- **Program-owner posture** treats the multi-objective report, the routing eval, the replacement-regression harness, the app-altitude latency report, and the per-feature cost report as the load-bearing decision artefacts a program owner defends at business review → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
- **`model-evaluation-engineer` (peer, level 30)** owns the model-altitude offline benchmarks (MMLU / GPQA / HumanEval / MMMU / etc.) and the MLPerf-shaped serving benchmarks. Chapter 07 defines what this module reads from the peer (the frontier-model quality profile, the serving latency benchmark curves) and what it hands back (the app-altitude quality gap, the cohort where the mid-tier model regressed, the streaming p95 the user actually sees).
- **`ai-infra-performance` (peer)** owns the serving fleet — kernels, batching, KV-cache management, GPU-hours-per-token, capacity planning. Chapter 07 defines the reconciliation contract: app-altitude latency reports back into the fleet's SLOs; fleet-level tail-latency changes flow into the app-altitude report as a lineage key.
- **The eval-data-platform slice** ([`mod-110`](../mod-110-eval-data-platform-slice)) is where the report reads from. Every report row in this module is a query over `eval_results`, `eval_runs`, `traces`, `spans`, `pricing_snapshots`, and the per-feature cost-centre roster. If the slice is not yet in place, the exercises can still run against a smaller CSV-plus-Postgres substitute — the shapes are the same.
