# mod-105 — RAG Evaluation at the Application Layer

**Estimated effort:** 12 hours

This is the fifth module of the AI Evaluation Engineer track. It takes the mod-101 SLIs, the mod-102 trace shape, the mod-103 trajectory-eval discipline, and the mod-104 judge pipeline, and specialises them for retrieval-augmented generation surfaces — chat assistants, search-shaped copilots, and document-QA surfaces where the reply is conditioned on retrieved context. The output is a slice-labelled RAG eval suite whose per-request scores mod-106 (release-gate at merge), mod-107 (online-eval on sampled traces), mod-108 (safety-side rubrics for grounded refusals), mod-109 (human review of judge disagreements on faithfulness / refusal calls), mod-110 (RAG-triad lineage on the eval-data platform), mod-111 (context-utilisation and top-k cost roll-ups), and mod-112 (release-gate architecture for RAG surfaces) all consume.

The module scopes to *application-layer* RAG eval — the RAG triad and its diagnostics, the retrieval-vs-generation attribution discipline, the product-shaped suite design with distractors / out-of-scope / hallucination-tempting prompts, and the noise-sensitivity / context-utilisation diagnostics that keep silent regressions from shipping. Retriever tuning, reranker choice, embedding-model selection, chunker design, and hybrid-retrieval architecture belong to the `rag-engineer` peer track; chapter 05 is the scope contract.

## Learning objectives

- Apply the **RAG triad** — faithfulness, answer relevancy, and the context precision / recall pair — using RAGAS, TruLens, and DeepEval and reason about each metric's documented failure modes.
- **Separate retrieval eval from generation eval** with recall@k, MRR, and hit rate on a labelled gold set; use fixed-context replay to isolate the generator; walk the attribution decision tree that resolves a triad regression to retrieval, generation, coupling, or joint.
- Design a **product-shaped RAG eval suite** for chat, search, or copilot surfaces with distractor documents, out-of-scope queries, and hallucination-tempting prompts — slice-labelled, real-query-sourced, and weighted by importance not frequency.
- Add **noise-sensitivity and context-utilisation diagnostics** (plus answer completeness and refusal quality) so a RAG regression is not silent when the triad aggregates are green.
- Coordinate with the **`rag-engineer` peer track**: this module owns the eval; retriever / reranker / chunker / index / corpus tuning belongs there. Chapter 05 is the scope contract and the handoff shape.

## Lecture chapters

1. [The RAG triad: faithfulness, answer relevancy, context precision and recall](01-rag-triad-metrics-and-failure-modes.md) — the four metrics, what each measures, what each requires (reference answer or not), the documented failure modes for each, the framework choice (RAGAS / TruLens / DeepEval), and how they attach to the mod-102 trace.
2. [Splitting retrieval and generation: attribution when a RAG regression fires](02-retrieval-vs-generation-attribution.md) — retrieval-only metrics (recall@k, MRR, hit rate) on the labelled gold set; fixed-context replay for the generator; the four-outcome attribution decision tree; per-slice reporting; a worked attribution.
3. [Designing a product-shaped RAG eval suite](03-product-shaped-suite-design.md) — real queries, real corpus, weighted-by-importance failure cases; the three signature slices (distractor documents, out-of-scope queries, hallucination-tempting prompts); harvesting from production; the slice tags on the trace.
4. [Noise sensitivity and context utilisation: diagnostics that keep regressions from going silent](04-noise-sensitivity-and-context-utilisation.md) — the four diagnostics (noise sensitivity, context utilisation, answer completeness, refusal quality) that sit next to the triad; a per-slice × per-metric dashboard; a worked silent-regression catch.
5. [The scope contract with the `rag-engineer` peer](05-rag-engineer-handoff.md) — what this module owns, what the peer owns, the boundary artefacts flowing both ways, the anti-patterns to reject, and the slice-labelled handoff template.

## Exercises

Each exercise builds on the last. Do them in order and keep the outputs — the artefacts (RAG-triad suite, attribution runbook, product-shaped gold set, diagnostic dashboard) are the deliverables mod-106 / mod-107 / mod-110 will assume you have.

1. [RAG triad with RAGAS and TruLens](exercises/exercise-01-rag-triad-with-ragas-and-trulens.md) — implement faithfulness / answer relevancy / context precision / context recall in both frameworks on the same dataset; report the per-metric agreement and pick one to keep.
2. [Retrieval vs generation attribution](exercises/exercise-02-retrieval-vs-generation-attribution.md) — build the retrieval-only metrics on a labelled gold set, the fixed-context replay, and walk two seeded regressions through the attribution decision tree.
3. [Product-shaped RAG suite with distractors](exercises/exercise-03-product-shaped-rag-suite-with-distractors.md) — author the labelled gold set with the three signature slices; report per-slice triad scores; document the harvesting and re-audit policy.
4. [Noise sensitivity and context utilisation](exercises/exercise-04-noise-sensitivity-and-context-utilisation.md) — wire the four diagnostics next to the triad; build the per-slice × per-metric dashboard; seed a silent regression and confirm the diagnostic paged before the triad did.

## Labs and quizzes

- Labs (see [`labs/`](labs)) build the end-to-end RAG eval harness that mod-106 wires into the merge gate and mod-107 runs against sampled production traces. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary — the four triad metrics and their failure modes, the four attribution outcomes, the three signature slices, the four diagnostics, and the peer-scope contract. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- **CI-gated eval** wires the RAG-triad suite and the per-slice acceptance thresholds into the merge and canary gates → [`mod-106-eval-gated-cicd`](../mod-106-eval-gated-cicd).
- **Online-eval and regression** runs the online-viable metrics (faithfulness, answer relevancy, context utilisation, refusal quality) on sampled production traces and pages on diagnostic-only regressions using the chapter-04 rules → [`mod-107-online-eval-and-regression`](../mod-107-online-eval-and-regression).
- **Safety and guardrail eval** reuses the refusal-quality rubric shape for the safety-family criteria on grounded surfaces → [`mod-108-app-safety-and-guardrails-eval`](../mod-108-app-safety-and-guardrails-eval).
- **Human review workflows** consume judge disagreements on faithfulness and refusal calls as the review queue → [`mod-109-human-review-workflows`](../mod-109-human-review-workflows).
- **Eval-data platform** stores the labelled gold set, the distractor / out-of-scope / hallucination-tempting slice labels, the replay sets, and the per-request `app.rag.*` lineage as first-class tables → [`mod-110-eval-data-platform-slice`](../mod-110-eval-data-platform-slice).
- **Cost / latency / quality trade-off** rolls up context utilisation and top-k against per-request cost and reply-quality → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- **Program-owner posture** treats the slice-weighted suite, the diagnostic dashboard, and the peer scope contract as first-class release-gate artefacts → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
- **Peer-track handoff to `rag-engineer`** for retriever / reranker / chunker / index / corpus tuning driven by the slice-labelled regression reports from chapter 05.
