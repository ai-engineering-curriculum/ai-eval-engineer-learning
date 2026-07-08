# mod-105-rag-eval-app-layer: RAG Evaluation at the Application Layer

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 12 hours

## Learning objectives

- Apply the RAG-triad (faithfulness, answer relevancy, context precision & recall) using RAGAS / TruLens / DeepEval and reason about each metric's failure modes
- Separate retrieval eval (recall@k, MRR, hit rate on a labelled gold set) from generation eval (faithfulness / answer relevancy) and diagnose which side of the pipeline a regression came from
- Design a product-shaped RAG eval suite for chat, search, or copilot surfaces with realistic distractor documents, out-of-scope queries, and hallucination-tempting prompts
- Add noise-sensitivity and context-utilisation diagnostics so a RAG regression is not silent
- Coordinate with the `rag-engineer` peer track: this module owns the eval; retrieval / rerankers / chunking depth belongs there

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
