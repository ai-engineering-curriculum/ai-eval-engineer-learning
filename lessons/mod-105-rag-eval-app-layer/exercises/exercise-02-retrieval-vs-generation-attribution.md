# exercise-02: Retrieval vs Generation Attribution

**Estimated effort:** 3 hours

## Objective

Build the **attribution machinery** from chapter 02 for your RAG surface: the retrieval-only metrics (recall@k, MRR, hit rate) computed against a labelled gold set with no LLM call, the **fixed-context replay** that isolates the generator, and the runbook that walks a triad regression through the decision tree to *retrieval / generation / coupling / joint*. Then seed two synthetic regressions (one retrieval, one generation) and confirm the runbook attributes each correctly with a paper trail.

By the end you will have a small attribution CLI you can run on any triad regression, a two-page runbook that a teammate can execute in ~15 minutes, and two attribution reports that resolve the seeded regressions to the correct side.

## Prerequisites

- Chapters 01 and 02 of this module.
- Exercise-01 completed — the RAG-triad implementation and the dataset.
- A **labelled gold set** for your surface with `query`, `relevant_chunk_ids`, `reference_answer` at minimum. Grow the 25–50-item dataset from exercise-01 to at least 100 items for this exercise; you will fill in the additional slice labels in exercise-03.
- A **retriever** you can point at the gold-set queries (production retriever behind a local dev shim is fine).
- A **generator** you can point at fixed retrieved contexts (i.e., you can bypass retrieval for the replay tests). If your surface's generator is tightly coupled to retrieval, factor the coupling out for this exercise.

## Requirements

Produce a single directory `mod-105/exercise-02/` in your working repo containing:

1. **`retrieval_metrics.py`** — computes recall@k (for k ∈ {1, 3, 5, 10}), MRR, hit rate@k, and precision@k for each query in the gold set. No LLM call. Reads `retrieved_chunk_ids` (from a live retriever run) and `relevant_chunk_ids` (from the gold set). Emits per-item and per-slice aggregates to `results-retrieval.csv`.
2. **`generator_replay.py`** — takes the current retriever's output for each gold-set query as a *frozen* context (`replay-context.jsonl`), and runs the current generator against it. Scores the reply with the exercise-01 faithfulness and answer-relevancy implementation. Emits per-item scores to `results-generator-replay.csv`. This is the isolate-the-generator side of the attribution tree.
3. **`retriever_replay.py`** — takes the current generator with a *pinned* prompt template and model snapshot, and runs the pipeline end-to-end against the gold-set queries. When the retriever changes, this is the delta run; when it doesn't, this is the current baseline. Emits per-item scores to `results-retriever-replay.csv`. This is the isolate-the-retriever side.
4. **`attribute.py`** — the CLI. Given two runs (baseline vs candidate), computes the delta on retrieval-only metrics, generator replay, retriever replay, and joint triad, and prints the attribution decision tree's verdict — one of `retrieval | generation | coupling | joint`. Include the "3–5 representative failing queries" printout with retrieved chunk ids and gold ids.
5. **`runbook.md`** — 300–500 words. The runbook a teammate follows in ~15 minutes when a triad regression pages. Sections:
   - The attribution decision tree from chapter 02 as a numbered list.
   - The three commands to run (retrieval metrics, generator replay, retriever replay).
   - How to read the four outcomes.
   - The escalation contract to the `rag-engineer` peer (link chapter 05's handoff template).
   - What to write in the postmortem.
6. **`seeded-regressions/`** — two seeded regressions with attribution reports:
   - **`retrieval-regression/`**: a synthetic retriever change that degrades retrieval quality on a slice (e.g., top-k reduced from 5 to 3, or a filter that excludes a fraction of the corpus). Include the before / after runs, the `attribute.py` output, and a 150-word report that names the slice, the metric that moved, and the peer handoff shape.
   - **`generation-regression/`**: a synthetic generator change that degrades faithfulness without touching retrieval (e.g., a prompt template edit that removes the "cite the retrieved context" instruction, or a temperature bump). Include the before / after runs, the `attribute.py` output, and a 150-word report that names the metric, the diff, and the fix.

## Starter guidance

- **Retrieval-only metrics are deterministic and cheap.** They should run in <10 seconds on 100 items — no LLM call, just set operations on chunk-id lists. If they're slower than that, your data plumbing has an issue, not the metrics.
- **Chunk ids must be stable** across the frozen contexts and the live retrieval. If your chunker regenerates ids on every rebuild, this exercise breaks — either freeze the index for the exercise or use a stable id derived from chunk content hash. Chapter 05's coordination with `rag-engineer` covers the chunk-id stability contract; for this exercise, note the choice.
- **The seeded regressions must be *actual* regressions on your metrics**, not just diff-visible changes. Test both regressions produce a triad-metric delta of at least 3–5 points on the slices they touch. If a seeded change doesn't move the metric, pick a more aggressive change.
- **The runbook is the deliverable**, not the CLI. The CLI is what makes the runbook fast. A runbook that says "read the CLI output and think about it" is not a runbook.
- **Include the per-slice denominators.** A retrieval-only metric on a 20-item multi-hop slice with 15 successes reads differently from the same rate on 3 successes out of 4. `attribute.py` should print the denominator for every metric it reports.

## Acceptance criteria

You are done when:

- `retrieval_metrics.py` computes recall@k / MRR / hit rate / precision@k on the gold set without an LLM call and reports per-slice aggregates with denominators.
- `generator_replay.py` and `retriever_replay.py` both run end-to-end and produce comparable per-item score CSVs.
- `attribute.py` takes two runs and prints the attribution verdict — `retrieval | generation | coupling | joint` — with the retrieval-only delta, the replay delta, the joint triad delta, and the 3–5 representative failing queries.
- The runbook is executable by a teammate in ~15 minutes on your surface without your commentary.
- Both seeded regressions produce reports that name the correct side (`retrieval-regression/` → retrieval verdict; `generation-regression/` → generation verdict), with per-slice evidence and the correct handoff shape.
- The exercise directory does not hard-code API keys; the retriever, generator, and judge are all pointed at pinned model / snapshot versions read from a shared config file.

## Stretch goals

- **Add a coupling regression to the seeded set.** Ship a retriever change *and* an unrelated prompt tweak together; confirm `attribute.py` returns `coupling` and the runbook resolves it to two owners.
- **Wire the attribution CLI to the mod-102 trace.** Read `app.rag.retrieved_chunk_ids` and `app.rag.gold_relevant_ids` from the trace store directly (rather than from a local CSV), so the CLI runs on real traffic rather than on the offline gold set. Confirm the online attribution matches the offline attribution on a shared subset.
- **Report the retrieval-only metrics as nDCG.** Add `sklearn.metrics.ndcg_score` to `retrieval_metrics.py` for graded-relevance labels (if your gold set supports them). Compare the nDCG-based ranking of retriever candidates against the MRR-based ranking.
- **Add a bootstrap CI on the per-slice attribution deltas.** A 3-point drop on a 20-item slice may not be significant; a 3-point drop on a 200-item slice is. Reuse `scipy.stats.bootstrap` from mod-104/chapter-04's calibration work.

## What this exercise does *not* cover

You are not designing the slice labels themselves (exercise-03). You are not adding the noise-sensitivity / context-utilisation diagnostics (exercise-04). You are not tuning the retriever (that is the `rag-engineer` peer's work; chapter 05). Stay narrow: retrieval-only metrics, two replays, one CLI, one runbook, two attribution reports.
