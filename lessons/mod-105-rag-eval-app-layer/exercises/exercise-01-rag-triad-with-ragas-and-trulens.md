# exercise-01: RAG Triad with RAGAS and TruLens

**Estimated effort:** 3 hours

## Objective

Implement the **RAG triad** — faithfulness, answer relevancy, context precision, context recall — in **both RAGAS and TruLens** against the same small labelled dataset on your RAG surface, and report the per-metric agreement between the two framework implementations. Pick one framework to keep for the surface and defend the pick.

By the end you will have two parallel implementations of the four metrics, a per-item results table for each, an agreement matrix between the two frameworks, and a bake-off note in the shape mod-107's online-eval loop will consume. This is the reference measurement the rest of the module builds on.

## Prerequisites

- Chapter 01 of this module.
- Chapters 01 (rubrics) and 03 (frameworks) of mod-104. The RAG-triad metrics are judge-scored under the hood; the mod-104 discipline on rubric anchors, judge model pinning, and prompt-hash logging applies here.
- Python 3.11+, `ragas`, `trulens`, and a frontier judge API key. Install fresh from the current docs (<https://docs.ragas.io/> and <https://www.trulens.org/>) — both libraries release frequently and rename APIs.
- A **small labelled dataset** — 25–50 items — from your RAG surface: `query`, `retrieved_contexts` (the ordered list of retrieved chunks, verbatim as your production retriever returns them), `answer` (the generator's reply), `reference_answer` (the gold answer). If you don't have retrieval / generation for the surface yet, use the sample dataset each framework ships (`ragas.testset` or the TruLens Colab notebook dataset) — note the substitution in the bake-off.

## Choose a surface and a dataset

Pick the same surface you used in mod-104/exercise-01 if it is a RAG surface. Otherwise pick one of: a document-QA assistant over a policy PDF, a code-QA copilot over an internal repo docs corpus, or a support-agent surface over a knowledge base. The scoring will run against 25–50 items; you can grow to a real gold set in exercise-03.

For each item you need:

- `query` — the user question.
- `retrieved_contexts` — the top-k chunks retrieval returned, as a list of strings.
- `answer` — the generator's reply.
- `reference_answer` — a hand-authored correct answer (for context recall and to reason about answer relevancy).

For at least 5 items, also include `reference_contexts` — the labelled relevant chunk contents (for the non-LLM context precision / recall implementation in RAGAS). If your dataset has chunk ids, include them too so the exercise-02 attribution work reads directly off the trace.

## Requirements

Produce a single directory `mod-105/exercise-01/` in your working repo containing:

1. **`dataset.jsonl`** — the 25–50 items, one JSON object per line, with `query`, `retrieved_contexts`, `answer`, `reference_answer`, optional `reference_contexts` and `chunk_ids`. Version-control this file — chapter 03 will grow the gold set from here.
2. **`ragas_run.py`** — implements the four triad metrics in RAGAS. Use the metrics from the current docs (as of writing: `Faithfulness`, `AnswerRelevancy`, `LLMContextPrecisionWithReference` or `ContextPrecision`, `LLMContextRecall`; confirm the current class names against <https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/>). Emit per-item scores and per-item rationales / claim lists to `results-ragas.csv`.
3. **`trulens_run.py`** — implements the same four metrics in TruLens. Use TruLens's feedback functions (`Groundedness`, `Relevance`, `ContextRelevance` for the triad — note that TruLens's stock triad is groundedness + context relevance + answer relevance; **context recall is not shipped as a top-level TruLens metric**, so for the recall metric implement it against RAGAS's shape and note the delta in the bake-off). Emit per-item scores to `results-trulens.csv`.
4. **`agreement.py`** — reads the two result CSVs and produces a 4-row × 2-metric agreement report:
   - Per metric, Spearman ρ between the RAGAS score and the TruLens score across items.
   - Per metric, mean absolute difference between the two scores (0–1 scale).
   - Per metric, count of items where the two frameworks disagree on the pass/fail side of a shared threshold (0.5 by default; adjust and document).
   Emit the report to `agreement.md`.
5. **`bakeoff.md`** — 400–600 words in three sections:
   - **Prompt-rendering diff.** For each metric in each framework, capture the actual judge prompt the framework emits (framework debug logging is your friend — both RAGAS and TruLens expose logging that shows the prompt). Note where the two frameworks decompose claims differently, where they use embedding similarity instead of a direct judge call (RAGAS's answer relevancy is embedding-based; TruLens's isn't), and where the anchor language differs.
   - **Score-agreement matrix.** The 4-row report from `agreement.py` written up as narrative. Read as: "how much does the framework choice move the metric before any prompt / model change." A metric where the two frameworks disagree materially (mean |Δ| > 0.15, or Spearman ρ < 0.5) is a metric where the framework choice is a design decision, not an implementation detail.
   - **Recommendation.** Which framework you would keep for this surface, and why. Grounded in: which metrics you actually need (recall requires RAGAS or a TruLens DIY), how naturally each wires to your team's trace backend (mod-102 chapter 03 — TruLens is trace-native; RAGAS is dataset-native), and how the anchor discipline from mod-104 rolls into each framework's rubric surface.
6. **`framework-tradeoffs.md`** — a short table (~10 rows) with per-framework properties from the current docs: reference-answer-required per metric, trace-native (yes/no), pytest-native (yes/no), judge-model pluggability, claim-decomposition available, embedding-based fallback, license. Fill from the current linked docs — do not invent rows.

## Starter guidance

- **Pin the same judge model and same temperature across both frameworks** for the primary run. Metric-level differences that are *not* about the framework's rubric or decomposition — differences that come from the judge model itself — are noise for this exercise. Both RAGAS and TruLens accept a judge LLM as a constructor argument.
- **Log the actual judge prompts.** RAGAS: enable the framework's prompt logging; TruLens: use the feedback function's `.run` result to inspect prompts. Save one sample rendered prompt per (metric × framework) to `sample-prompts/`. These are what your bake-off argument is grounded in.
- **RAGAS's answer relevancy is embedding-based by default.** It generates a paraphrased-question from the reply and computes cosine similarity to the original question. TruLens's answer-relevance is a direct judge call. This is the largest single implementation difference between the two on the triad; call it out explicitly in the bake-off.
- **Context recall is the metric with the largest framework gap.** TruLens does not ship a stock recall metric because it does not assume reference answers. If you keep TruLens, plan to implement recall yourself (or run RAGAS just for that one metric); make the trade-off explicit in the recommendation.
- **Version-control the dataset.** The rest of the module's exercises grow it. Do not silently regenerate.
- **No hard-coded API keys.** Read from environment variables. Same for judge model snapshot names — pin a snapshot in a shared config file.

## Acceptance criteria

You are done when:

- `dataset.jsonl` has 25–50 real (or realistic) items with the required fields.
- `ragas_run.py` and `trulens_run.py` both execute end-to-end and emit per-item scores to their respective CSVs.
- All four triad metrics are scored in RAGAS. In TruLens, the three natively-shipped metrics are scored; the context-recall gap is documented in the bake-off with the implementation choice (skip / DIY / use RAGAS for recall only).
- `agreement.md` reports Spearman ρ, mean |Δ|, and disagreement count per metric.
- `bakeoff.md` cites the actual rendered prompts (saved to `sample-prompts/`), reads the agreement matrix as data, and lands on a recommendation grounded in the surface's real trace-backend and rubric-discipline needs.
- `framework-tradeoffs.md` is filled from the current linked docs, no invented rows.
- Both frameworks were pinned to the **same judge model and temperature**; deltas from framework-side differences (embedding vs judge, decomposition granularity, anchor language) are explicit, not silent.

## Stretch goals

- **Add DeepEval as a third leg.** Wire the same four metrics through DeepEval's contextual metrics (`FaithfulnessMetric`, `AnswerRelevancyMetric`, `ContextualPrecisionMetric`, `ContextualRecallMetric`). Extend the agreement matrix to 3×3. This gives you the shape you'll need if the mod-106 CI gate ends up pytest-native.
- **Wire the RAGAS run behind a real trace backend.** If your team runs Phoenix / Langfuse / LangSmith / Braintrust, configure the RAGAS integration (or a small OpenInference adapter) to attach per-metric EVALUATOR spans to the request traces. Confirm the trace viewer surfaces the score + rationale on the span.
- **Compute the mod-104/chapter-02 bias controls on the two most-judge-heavy metrics (faithfulness, answer relevancy).** Run each judge with a swapped position of the reference / candidate blocks in the prompt, or with a length-normalisation adjustment. Report the residual bias on each metric per framework.
- **Add the non-LLM RAGAS metrics.** `NonLLMContextPrecisionWithReference` and `NonLLMContextRecall` (<https://docs.ragas.io/>) score without a judge call at all — deterministic, cheap, viable in CI. Compare their per-item scores against the LLM-judge versions; the gap is a signal for chapter 02.

## What this exercise does *not* cover

You are not designing the labelled gold set at scale (exercise-03). You are not diagnosing which side of the pipeline a regression comes from (exercise-02). You are not adding the noise-sensitivity / context-utilisation diagnostics (exercise-04). Stay narrow: the four triad metrics, two frameworks, 25–50 items, one bake-off.
