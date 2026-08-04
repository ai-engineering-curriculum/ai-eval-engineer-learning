# Splitting Retrieval and Generation: Attribution When a RAG Regression Fires

## Motivation

The RAG-triad dashboard drops. Faithfulness slid three points, answer relevancy is flat, context precision is soft. Every team on the on-call rotation has the same first question: *"is this a retrieval regression or a generation regression?"* — because the fixes are owned by different people and cost different amounts of engineering time. The `rag-engineer` peer tunes the retriever, the reranker, and chunking; the LLM-application engineer tunes prompts, tools, and the generator model. If you page the wrong side, the actual regression goes un-triaged for however long it takes to walk it back.

This chapter is the diagnostic discipline that turns *"RAG got worse"* into *"retrieval regressed on out-of-scope queries by 8% recall@5"* or *"the generator started paraphrasing chunks in a way that scores as unsupported."* The tool is a small set of **retrieval-only metrics** (recall@k, MRR, hit rate on a labelled gold set) computed independently of the generator, paired with a **fixed-context replay** that isolates the generator from retrieval. When both are wired, every triad regression resolves to one of three buckets: retrieval, generation, or joint.

## Core concepts

### Why the triad alone doesn't attribute

The four triad metrics from chapter 01 are all *outputs of the whole pipeline*. Context precision looks upstream, but the "precision" it measures is over whatever chunks the retriever surfaced on this specific query — a retrieval regression that changes *which* chunks are surfaced changes the numerator and denominator simultaneously; a generation regression that changes claim decomposition changes the metric with retrieval untouched.

Two hypothetical regressions that look identical on the triad:

- Retrieval starts returning off-topic chunks for a slice of queries. Context precision drops. The generator, staring at off-topic chunks, either hallucinates around them (faithfulness drops) or answers around them (answer relevancy drops). Every triad metric moves.
- The generator model is upgraded, tokenises numbers differently, starts paraphrasing retrieved figures instead of quoting them. Faithfulness's claim-decomposer marks the paraphrases as unsupported. Faithfulness drops. Context precision is unchanged in raw ranking but the ordering-aware aggregate shifts because the reply now leans on lower-ranked chunks. Two triad metrics move.

Same dashboard shape, opposite fixes. The triad tells you *something moved*; it does not tell you *what*.

### Retrieval-only metrics: recall@k, MRR, hit rate

These are the metrics computed against the retrieved chunk-id list and a gold set, with no LLM call and no reference to the reply. They are the pure-retrieval SLIs the `rag-engineer` peer owns and that you own as the eval engineer downstream.

**Recall@k.** Of the reference chunks required to answer the question (from the labelled gold set), what fraction did the retriever surface in its top-k. Formally, `|retrieved_top_k ∩ relevant| / |relevant|`. This is the "did the passage even show up" metric — the load-bearing question for whether the generator had a chance.

**MRR (Mean Reciprocal Rank).** For each query, take the rank of the first relevant chunk in the returned list, take its reciprocal (`1/rank`, or 0 if no relevant chunk was returned), and average across queries. Sharp for surfaces where the generator uses the top result more heavily than the rest — chat surfaces, single-answer surfaces. See TREC's MRR definition (<https://trec.nist.gov/pubs/trec21/appendices/measures.pdf>).

**Hit rate.** The fraction of queries where at least one relevant chunk appears in the top-k. Coarser than recall@k, but a more forgiving signal on multi-chunk-answer queries. Useful as a "did retrieval find anything relevant at all" cross-check.

**Precision@k and nDCG.** Precision@k is `|retrieved_top_k ∩ relevant| / k` — useful for measuring "how much of the context window is being wasted on irrelevant chunks." nDCG (normalised discounted cumulative gain) weights each rank position by an information-gain-shaped function and normalises by the ideal ranking; use it when relevance labels are graded (not binary) and rank position matters, and don't reinvent it — RAGAS and BEIR both ship implementations, and `sklearn.metrics.ndcg_score` (<https://scikit-learn.org/stable/modules/generated/sklearn.metrics.ndcg_score.html>) is the reference for the ranking-metric definition.

All four are computed from the RETRIEVER span's chunk-id list against a labelled gold set. No judge required. They are cheap, deterministic, and can be run in CI on every commit (mod-106). The RAGAS non-LLM context precision / recall implementation uses these directly (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/non_llm_context_precision/>).

### The labelled gold set that makes retrieval eval possible

Retrieval metrics are only as sharp as the labels. A useful gold set for retrieval eval has three columns:

- **`query`** — the user input, verbatim or lightly normalised.
- **`relevant_chunk_ids`** — the set of chunk ids that would let a competent human answer the query. Multi-chunk answers get multiple ids.
- **`answer`** — the reference answer (optional but strongly preferred — RAGAS's `LLMContextRecall` and DeepEval's `ContextualRecallMetric` both use it).

Sizing depends on the surface: 100–300 items is enough to see a retrieval regression on the aggregate; 500–1000 is what you want if you also want *per-slice* attribution (out-of-scope, multi-chunk, distractor-heavy — chapter 03's shape). Below 100 the aggregate is too noisy to catch anything but the largest regressions.

The gold set lives next to the rest of the eval data, versioned. mod-110 covers the storage; for the purposes of this chapter, treat it as a source-controlled JSONL with the three columns and a `version` header, and re-label at least the top-N most-frequently-hit queries whenever the corpus is re-chunked (chapter 05 — chunking changes chunk ids, and stale ids will make recall look worse than it is).

### Fixed-context replay: isolating the generator

Retrieval-only metrics isolate the retriever. To isolate the generator, do the reverse: hand it a *fixed* retrieved context and score the reply.

The pattern:

1. Freeze a small subset of the offline gold set (~50–200 items) as the **generator replay set**. Each item has the query and a *fixed* retrieved context — the retrieval the current-generation retriever returns *today*, saved verbatim.
2. When you release a new generator (prompt, model, tool config), replay the frozen retrieval through the new generator and score the reply with the RAG triad's *generation* metrics — faithfulness and answer relevancy. Skip context precision and recall (they're constant by construction on this replay).
3. Compare the new generator's replay scores to the current generator's replay scores on the *same* frozen context. Any delta is a generation-only regression, by construction. Retrieval is out of the equation.

The same pattern with the roles swapped isolates the retriever:

1. Freeze a **retriever replay set** where each item has the query and a *fixed* generator (a specific prompt template and a pinned model snapshot).
2. When you release a new retriever, replay the query through the new retriever and then through the fixed generator, and score with the triad. Any delta is a retrieval-driven regression.

Both replay sets live next to the labelled gold set. Both are re-frozen on a schedule (typically each release), so "current retrieval today" doesn't rot.

### The attribution decision tree

Given a triad-metric regression, walk the tree:

1. **Run the retrieval-only metrics on the labelled gold set.**
   - If **recall@k / MRR / hit rate dropped**: retrieval regression. Escalate to `rag-engineer` peer with the specific slice that dropped.
   - If retrieval-only metrics are flat: retrieval-only regression is unlikely; continue.
2. **Run the generator replay on the frozen context.**
   - If faithfulness / answer relevancy dropped on the replay: generation regression. Own it — pin the incoming generator diff and bisect.
   - If replay scores are flat: the generator on frozen context is fine.
3. **If both are flat but the joint triad still shows a regression**: it's a *coupling* regression — the retriever's outputs changed in a way that only the current generator is sensitive to, or vice versa. Two subcases:
   - **New retrieval, old generator prompt.** The retriever now surfaces chunks the prompt template's instructions don't handle well (different chunk length, different section headers). Fix by adjusting the prompt template with the retrieval owner, or by moving prompt template to handle whatever new chunk shape the retriever emits.
   - **New generator model, old retrieval.** The generator now paraphrases in ways the judge's claim-decomposer misreads. Fix by re-anchoring the faithfulness rubric or by pinning the claim-decomposer to a stable snapshot.
4. **If retrieval-only dropped *and* replay dropped**: joint regression; likely a coincident release. Handle each independently.

Every step of the tree writes a slice of the answer back into the postmortem: "recall@5 dropped 6 points on multi-hop queries; generator replay was flat; escalated to rag-engineer with the multi-hop slice." The tree is 3–4 commands to run, not an investigation.

### Slicing the metrics so attribution is per-slice, not per-corpus

An aggregate regression can hide a slice regression that inverts (retrieval improved on head queries and collapsed on tail queries; the mean is unchanged). Every metric in this chapter should be logged with a `slice` attribute so the postmortem can group by slice:

- **Query type / intent.** Head vs tail queries, single-fact vs multi-hop, in-scope vs out-of-scope (chapter 03).
- **Retrieval difficulty.** Queries where recall@1 was <1 (multi-chunk answers) vs recall@1 = 1 (single-chunk answers).
- **Generator input shape.** Short vs long retrieved context; retrieval with vs without distractors (chapter 03); reranker-on vs reranker-off (chapter 05 boundary).

If the eval only reports an aggregate score, you lose the slice signal. If it reports per-slice with denominators, the attribution decision tree above resolves in two commands.

### The `app.rag.*` attribution attributes on the trace

Continuing chapter 01's mod-102 extension list, the attribution work adds:

- `app.rag.retrieved_chunk_ids` — the ordered chunk-id list retrieval surfaced. Populates recall@k / MRR / precision@k directly.
- `app.rag.gold_relevant_ids` — the labelled relevant ids for this query (only on the labelled gold set traces; empty otherwise).
- `app.rag.slice` — the slice tag(s) for this query (`intent=multi-hop,scope=in-scope,distractor=high`).
- `app.rag.replay_set` — one of `none | generator | retriever`; identifies traces that came from a replay so they don't pollute the online-eval aggregate.
- `app.rag.generation_delta_vs_replay` — for generator replay traces, the score delta vs the frozen baseline generator on the same context.

The retrieval-only metrics can be computed from `app.rag.retrieved_chunk_ids` + `app.rag.gold_relevant_ids` with a one-line aggregation — no LLM call, no judge, no framework. That is the point: attribution should be cheap.

### A worked attribution: 8-point faithfulness drop

Concrete scenario. On Monday, aggregated faithfulness on the multi-hop slice dropped 8 points. Attribution:

1. **Retrieval-only on the gold set for the multi-hop slice.** Recall@5 is flat at 0.72. MRR is flat at 0.61. Retrieval is not the cause on this slice.
2. **Generator replay on the frozen multi-hop context.** Faithfulness on the replay dropped 7 points. The regression is on the generator.
3. **Bisect the generator diff.** The prompt template was updated Friday to add a "cite section numbers when available" instruction. The generator now often paraphrases section text in the reply without quoting; the faithfulness judge, seeing a claim that says "§4.2 requires X" and a chunk that says "clause 4.2 shall require X", marks it borderline unsupported — enough to shift the aggregate.
4. **Fix**: adjust the faithfulness rubric anchor to accept close paraphrase when the section number matches, *and* soften the prompt template instruction to say "quote or cite section numbers." Add the paraphrase case to the calibration set (chapter 04 of mod-104).

Every step of the tree wrote to the postmortem. No paging the wrong team. No "we don't know if it's retrieval." The whole exercise takes an afternoon because the metrics were logged per-slice with the mod-102 attributes.

## Summary

- The **RAG triad** measures whole-pipeline outputs; it does not, on its own, attribute a regression to retrieval vs generation.
- **Retrieval-only metrics** — recall@k, MRR, hit rate, precision@k — run against a labelled gold set with no LLM call. They isolate the retriever and are the `rag-engineer` peer's primary SLI.
- **Generator replay** on frozen retrieval isolates the generator; retriever replay through a fixed generator isolates the retriever.
- The **attribution decision tree** (retrieval-only → generator replay → coupling case) turns a triad regression into a specific owned fix in 3–4 commands.
- **Slice every metric** — intent, retrieval difficulty, generator input shape. Aggregate-only reporting hides slice regressions that invert.
- Attribute the extra `app.rag.*` attributes onto the mod-102 trace so the whole tree runs off the trace store with no additional plumbing.

Chapter 03 walks how to design the *dataset* — the labelled gold set, the replay sets, the sliced coverage — that gives the metrics in this chapter something meaningful to run against.
