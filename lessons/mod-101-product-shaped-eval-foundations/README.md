# mod-101 — Product-Shaped Evaluation Foundations: From Spec to Eval Plan

**Estimated effort:** 10 hours

This is the first module of the AI Evaluation Engineer track. It builds the vocabulary and the artefact — the two-page eval plan — that the rest of the modules refine, wire into CI, run on live traffic, and defer intelligently to peer specialists. Nothing later in the track works if this step is skipped: `mod-102` (traces) records against SLIs you have not defined, `mod-106` (CI gates) enforces thresholds that were never registered, and `mod-107` (online eval) samples for regressions against criteria that were never falsifiable to start with.

## Learning objectives

- Read a product spec and derive concrete eval criteria — task success, groundedness, safety, cost, latency — that map to product SLOs.
- Decide what to measure offline (regression suite), what to measure online (sampled trace eval), and what to defer to the model-eval owner (statistical depth).
- Structure a per-surface eval plan using ISO/IEC 25059 quality dimensions as a checklist and NIST AI RMF Measure as a scope frame.
- Recognise the failure modes of leaderboard-shaped eval when it collides with a real product surface, and know when to reject a proposed benchmark.
- Write a two-page eval plan a product manager, an ML engineer, and a governance owner can all sign off on.

## Lecture chapters

1. [What "product-shaped" evaluation means](01-what-product-shaped-eval-means.md) — surface, altitude, the five criteria families, SLI/SLO/SLA, and the peer-owner map.
2. [From product spec to eval criteria](02-spec-to-criteria.md) — tag the spec for four claim types; decompose to criterion → SLI → SLO; apply the "so what" test.
3. [Offline, online, and deferred](03-offline-online-and-deferred.md) — the four-step decision rule; per-family placement defaults; sampling and cost budgets.
4. [ISO/IEC 25059 and NIST AI RMF as checklists](04-iso-25059-and-nist-ai-rmf.md) — the per-surface completeness pass and the program-level scope frame.
5. [When leaderboards collide with real product surfaces](05-leaderboard-vs-product-eval.md) — eight leaderboard failure modes; the eight-question rejection template; adopt-as-shape-template pattern.
6. [Writing the two-page eval plan](06-writing-the-eval-plan.md) — the eight-section template, minimum bars, anti-patterns, and a worked example.

## Exercises

Each exercise builds on the last. Do them in order; keep the outputs — later modules will assume you have them.

1. [Spec-to-criteria decomposition](exercises/exercise-01-spec-to-criteria-decomposition.md) — pick a surface, tag the spec, build the criteria table.
2. [Offline vs online vs deferred](exercises/exercise-02-offline-vs-online-split.md) — place every row of the criteria table, design the offline suite and online loop budgets.
3. [ISO/IEC 25059 and NIST AI RMF checklists](exercises/exercise-03-iso-25059-and-nist-ai-rmf-checklist.md) — completeness pass; the diff report often adds real rows to the criteria table.
4. [Leaderboard vs product-eval audit](exercises/exercise-04-leaderboard-vs-product-eval-audit.md) — audit three public benchmarks with the eight-question rejection template; write the stakeholder memo.

## Labs and quizzes

- Labs (see [`labs/`](labs)) wire the produced plan into a running fixture harness and a preflight CI check that `mod-106` builds on. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary and the decision rules. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- Trace instrumentation that records the SLIs you defined here → [`mod-102-trace-instrumentation`](../mod-102-trace-instrumentation).
- Trajectory / tool-call evaluation shaped by chapter 03's offline / online split → [`mod-103-trajectory-and-tool-eval`](../mod-103-trajectory-and-tool-eval).
- LLM-as-judge rubric design and bias control for the judge-scored SLIs above → [`mod-104-llm-as-judge-in-product`](../mod-104-llm-as-judge-in-product).
- RAG-triad on faithfulness / relevancy / context precision & recall → [`mod-105-rag-eval-app-layer`](../mod-105-rag-eval-app-layer).
- CI wiring for the offline suite and the merge / canary gates → [`mod-106-eval-gated-cicd`](../mod-106-eval-gated-cicd).
- Sampled-trace eval loop, drift detection, canary and shadow → [`mod-107-online-eval-and-regression`](../mod-107-online-eval-and-regression).
- Safety and guardrail-effectiveness measurement on the surface → [`mod-108-app-safety-and-guardrails-eval`](../mod-108-app-safety-and-guardrails-eval).
- Human review workflows behind the "deferred to human" rows → [`mod-109-human-review-workflows`](../mod-109-human-review-workflows).
- Eval-data-platform where fixtures, results, and lineage live → [`mod-110-eval-data-platform-slice`](../mod-110-eval-data-platform-slice).
- Cost / latency / quality trade-offs across model tiers → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- Owning the whole program (release-gate architecture, delegation contracts, build-vs-buy, incident-driven investment) → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
