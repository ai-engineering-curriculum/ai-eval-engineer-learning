# The Scope Contract with the `rag-engineer` Peer

## Motivation

This module owns the RAG *evaluation*. It does not own the retriever, the reranker, the embedding model, the chunker, or the index. Those belong to the `rag-engineer` peer track — a separate role with its own curriculum, its own architectural depth, and its own release cadence. When the two roles' scopes bleed together, the eval engineer starts tuning a chunker to make their metrics look better and the RAG engineer starts changing metric definitions to make their retriever look better. Both moves compromise the eval as a defensible measurement, and both are avoidable with a clean scope contract.

This chapter is the contract. What this module owns, what the `rag-engineer` peer owns, what artefacts flow across the boundary, and how a regression that spans the boundary gets resolved. The goal is that when an eval metric moves, the answer to *"who fixes this"* is a one-line lookup, not a multi-team meeting.

## Core concepts

### What this module owns

The eval engineer's scope for RAG:

- **The metric definitions.** Which RAG-triad metrics, which diagnostic metrics, how each is computed, what the anchors are, what the framework is (chapters 01, 04).
- **The eval suite.** The labelled gold set, the slice tags, the replay sets, the acceptance thresholds (chapter 03).
- **The attribution machinery.** The retrieval-only metrics on the gold set, the fixed-context replay, the decision tree that resolves a triad regression to retrieval / generation / joint (chapter 02).
- **The rubrics.** The faithfulness / answer-relevancy / refusal-quality rubrics, calibrated against a human gold set (mod-104/chapter-04).
- **The trace attribution.** The `app.rag.*` extensions on the mod-102 trace, the EVALUATOR spans that carry judge scores, the slice tags that drive per-slice aggregation.
- **The paging policy.** What thresholds and holdovers page which team on which regression (mod-107).
- **The release-gate criteria for the eval axis.** What has to pass for a candidate model / prompt / retriever to ship (mod-106, mod-112).

### What the `rag-engineer` peer owns

Not exhaustive — verify against the peer's curriculum — but the working boundary:

- **The retriever.** Vector store choice, hybrid retrieval design (BM25 + dense + sparse), embedding model choice, top-k tuning, filter / metadata design.
- **The reranker.** Cross-encoder choice, reranker model choice, reranker top-k, reranker calibration, cascaded reranking.
- **The chunker.** Chunk size, chunk overlap, semantic vs syntactic chunking, chunk metadata schema, chunk id stability across re-index.
- **The indexing pipeline.** Ingest, deduplication, freshness / TTL, index rebuild cadence.
- **The corpus.** Which documents live in the retrievable corpus, at what version, with what metadata.
- **The retrieval-side prompt hooks.** Query rewriting, HyDE-style expansion, self-query, multi-query fanout.
- **The retrieval SLIs.** Latency percentiles on the retriever, index freshness lag, index size.

### The boundary: what flows across

Two directions.

**Eval → RAG peer (what you send them):**

- **Slice-labelled regression reports.** When the attribution tree in chapter 02 lands on the retrieval side, the report they receive names the slice (`multi_hop`, `distractor_heavy`, `out_of_scope`), the metric that dropped (recall@5, MRR), the magnitude, the sample size, and 3–5 representative failing queries with their retrieved chunk ids. Not "retrieval regressed" — that's not actionable.
- **Gold-set updates.** When the labelled gold set changes (new queries added, distractor labels revised, chunk-id remap after re-index), notify. The `rag-engineer` peer runs their own offline retrieval eval against your gold set; they need to know when it changes.
- **Diagnostic thresholds.** Noise sensitivity and context utilisation thresholds you consider "healthy" for the surface, so the peer knows what shape of retriever / reranker they are optimising for.
- **The chunk-id ↔ metric mapping.** So a chunker change that alters chunk boundaries can be tested against the current gold set before it ships.

**RAG peer → Eval (what they send you):**

- **Corpus / index change notices.** Ahead of a chunker or re-index change, the peer flags that chunk ids will change. You re-audit the gold set labels before the change lands.
- **Retriever configuration snapshots.** Which retriever / reranker / chunker version is running in production and in the offline pipeline the gold set is scored against. Version-controlled config that the mod-102 RETRIEVER span records on `app.rag.retriever_version`.
- **Retrieval SLI dashboards.** Latency, index freshness, top-k distributions — signals the eval engineer references when a triad regression coincides with a retrieval infra event.
- **Retriever replay sets.** When they want to A/B a new retriever, they hand you the two chunk-id lists per query (baseline retriever vs candidate) and you run the fixed-generator replay from chapter 02.

The specific artefact names are per-team. What matters is that both directions are named artefacts, not ad-hoc slack messages, and both are versioned.

### Regressions that span the boundary: who drives the resolution

Every triad regression starts with the attribution tree from chapter 02. The tree resolves to one of four outcomes:

1. **Retrieval-only.** Recall@k / MRR / hit rate moved; fixed-generator replay is flat. Owned by `rag-engineer` peer. You hand off with the slice-labelled report and observe from the sidelines.
2. **Generation-only.** Retrieval-only metrics flat; fixed-context replay dropped. Owned by this module (with the LLM-application engineer for prompt / model fixes). Peer is informed but not on the hook.
3. **Coupling.** Both flat individually; joint moved. Both owners are on the hook. Someone has to drive — default to whoever released last; escalate to a joint review otherwise.
4. **Both moved independently.** Coincident releases; each side handles its own; explicitly agree on the release ordering to avoid re-testing.

The one anti-pattern to avoid: eval engineer *fixing the retriever* by tuning the eval to accept the regressed behaviour. If the metric moved and the fix is inside the eval definition, that's a real change to the definition — it needs its own approval, a re-calibration (mod-104/chapter-04), and a note in the release-gate spec (mod-112). Silently loosening a threshold or dropping a slice from the aggregate is how the eval becomes theatre.

The mirror anti-pattern from the peer's side: `rag-engineer` peer *fixing the eval* by hand-tuning retriever behaviour to specific queries in the gold set. Two rules to keep this honest:

- The gold set is fixed for a release; the peer optimises against a *disjoint* dev set of the same shape.
- Periodically refresh the gold set with new production-harvested queries (chapter 03), so per-query overfitting decays out of relevance.

### Prompts that touch retrieval: the shared-ownership grey area

Some prompt fragments live in both worlds:

- **Query rewrite / expansion prompts** (HyDE, multi-query, self-query). These are retrieval-side; the peer owns the prompt but the eval scores their impact on retrieval-only metrics.
- **Retrieved-context-conditioning prompts.** "Answer the user's question using only the following retrieved documents…" — generation-side; the LLM-application engineer owns the prompt.
- **Rerank-time judgement prompts.** If the reranker is LLM-based, the reranker's prompt is retrieval-side.

The rule: if the prompt changes what *retrieval* returns, it's the peer's; if the prompt changes what the *generator* does with retrieval, it's the LLM-application engineer's (informed by this module's rubrics). The mod-102 span shape lets you tell them apart — the query-rewrite prompt is on the RETRIEVER span or one of its children; the answer prompt is on the LLM span.

### Peer curriculum topics this module deliberately does not teach

- **Chunking strategy depth.** Semantic chunking vs syntactic, chunk-size ablations, overlap ablations. `rag-engineer` peer's curriculum.
- **Reranker choice and cascaded reranking.** Cross-encoders vs bi-encoders, LLM rerank vs specialised reranker, cost / latency trade-offs. Peer.
- **Hybrid retrieval design.** BM25 + dense + sparse blending, RRF vs learned fusion. Peer.
- **Embedding model selection.** OpenAI vs Cohere vs open-source, dimensionality trade-offs, matryoshka embeddings. Peer.
- **Vector store operations.** Choice of DB, replication, index rebuild, backup, autoscaling. Peer.
- **Ingest pipeline.** Document extraction, deduplication, chunker orchestration. Peer.

If a question hits any of the above and it is not the *eval-and-attribution* framing of it, the answer is "the `rag-engineer` peer has it." This module measures the outcomes of those choices; it does not make them.

### Cross-references to the peer's shape

- The peer's role catalogue and curriculum live under `rag-engineer` in the roles registry — verify the exact path before citing.
- The peer's release cadence and their retrieval SLI dashboards are the source-of-truth references you cite in a slice-labelled regression report.
- The peer's chunker-change process is the source-of-truth trigger for a gold-set chunk-id audit on your side.

<!-- needs-research: on the next research cycle, confirm the `rag-engineer` role's curriculum path in this org's roles registry and add a direct link here. If the role has canonical documentation on its retriever / reranker / chunker SLIs, cross-link. -->

### The handoff document template

Every retrieval-side regression report you send the peer follows a short template:

```
Regression: <metric> on <slice>, <magnitude>, <direction>
Sample size: <n items scored>, per-slice
Timeframe: <detection window>
Attribution: <retrieval-only|generation-only|coupling|both>
Evidence:
  - retrieval-only metrics: recall@5 = <before → after>, MRR = <before → after>
  - fixed-context replay: faithfulness delta = <before → after>
  - triad metrics: faithfulness, answer relevancy, ctx precision, ctx recall = <values>
Representative queries (3–5):
  - <query>, retrieved chunk ids, gold relevant chunk ids, current reply
Suspected drivers (informed guess, not diagnosis):
  - <hypothesis>
Requested response:
  - <acknowledge by, first analysis by, patch or rollback by>
```

Every field is either populated from the mod-102 trace directly or from a canned per-slice aggregation. The peer receives a report they can act on the same day, not a paragraph of narrative to unpack.

## Summary

- The `rag-engineer` peer owns **retriever / reranker / chunker / index / corpus**; this module owns **metrics, rubrics, slice design, attribution, and paging**.
- The boundary is two-directional and named: **slice-labelled regression reports** flow to the peer; **corpus / retriever / chunker change notices** flow back. Both are versioned artefacts, not slack threads.
- Every triad regression starts with the chapter-02 attribution tree. The output is one of four outcomes; each has a named driver.
- Anti-patterns to reject: eval engineer *fixing retrieval by tuning the eval*, RAG engineer *fixing the eval by hand-tuning to gold-set queries*. Both compromise the measurement.
- Prompt ownership follows the mod-102 span shape: retrieval-side prompts are the peer's; generator-side prompts are the LLM-application engineer's (with this module's rubrics).
- Depth on chunking, rerankers, embeddings, hybrid retrieval, ingest, and vector stores lives in the peer's curriculum. This module measures outcomes.

The module deliverables that flow out of this chapter — the gold set with slice tags, the attribution decision tree runbook, the handoff template, the rubric documents — are the artefacts the release-gate architecture in mod-112 assembles and the on-call playbook in mod-107 references.
