# quiz-01: RAG Evaluation at the Application Layer

**Estimated time:** 15–20 minutes
**Covers:** chapters 01–05 of this module.

Answer each question, then check the key at the bottom. Explanations cite the chapter the answer comes from so you can re-read the relevant section if an answer surprised you.

---

## Questions

### 1. Chapter 01 decomposes a RAG request into three legs and attaches a triad metric to each. Which pairing matches the chapter's table?

- A. Faithfulness ↔ retrieval precision; answer relevancy ↔ retrieval coverage; context precision ↔ generator groundedness; context recall ↔ question alignment.
- B. Context precision ↔ retrieval relevance; context recall ↔ retrieval coverage; faithfulness ↔ reply grounded in retrieved chunks; answer relevancy ↔ reply addresses the user's question.
- C. All four metrics measure generation; retrieval is scored out of band.
- D. Faithfulness and answer relevancy both measure retrieval; context precision and recall both measure generation.

### 2. Chapter 01 is explicit about which triad metrics are viable as online-eval SLIs on unlabelled production traffic. Which pair matches the chapter?

- A. Context precision and context recall — both score the retriever directly.
- B. Faithfulness and answer relevancy — faithfulness is scored against the retrieved chunks, answer relevancy against the user question; neither requires a gold answer.
- C. Only context recall; everything else needs a judge.
- D. None of them — every triad metric requires a reference answer.

### 3. A support-agent RAG reply says *"the policy allows this"* and the retrieved chunk says *"§4.2 permits this action."* Two faithfulness judges produce different verdicts: one marks supported, one marks unsupported. Chapter 01 names this failure mode and prescribes a fix. Which pair is right?

- A. Claim-decomposition failure; fix by running the judge at higher temperature.
- B. Paraphrase-vs-entailment slack; fix by pinning the entailment threshold through framework config or anchor language ("supported = a competent reader would accept the retrieved passage as the source").
- C. Judge over-crediting authoritative voice; fix by truncating the reply.
- D. Out-of-scope handling; fix by returning `not_applicable`.

### 4. Chapter 01 compares RAGAS, TruLens, and DeepEval on the four triad metrics. Which statement matches the chapter's framework table?

- A. RAGAS ships all four metrics; TruLens does not ship context recall as a top-level triad metric (because it does not assume reference answers); DeepEval is rubric-driven and pytest-native.
- B. TruLens ships all four; RAGAS skips faithfulness; DeepEval is deprecated.
- C. All three frameworks implement the four metrics identically and interchangeably — framework choice is purely cosmetic.
- D. Only DeepEval supports the triad; the other two frameworks do not.

### 5. Chapter 02 argues that the RAG triad alone does **not** attribute a regression to retrieval vs generation. What is the chapter's core reason?

- A. The triad metrics are all outputs of the whole pipeline, so a retrieval change and a generation change can both move the same set of triad metrics and look identical on the dashboard.
- B. The triad metrics are too noisy to detect any regression.
- C. The triad is only scored offline and cannot see online traffic.
- D. The triad requires a judge, which hides the retrieval signal by construction.

### 6. Chapter 02 names three "retrieval-only" metrics that run against the labelled gold set with no LLM call. Which triple matches?

- A. Faithfulness, answer relevancy, context precision.
- B. Noise sensitivity, context utilisation, answer completeness.
- C. recall@k, MRR (mean reciprocal rank), and hit rate@k — plus precision@k and nDCG as additional ranking-aware variants.
- D. BLEU, ROUGE, and METEOR.

### 7. Chapter 02's **fixed-context replay** is the pattern that isolates the generator. Which description matches the chapter?

- A. Replay the same user query through a new retriever and the current generator, then score the triad.
- B. Freeze the retrieval output (today's top-k verbatim) for a small subset of the gold set; when a new generator ships, replay the frozen context through the new generator and score with faithfulness and answer relevancy. Any delta is a generation-only regression by construction.
- C. Replay last week's traffic sample through today's full pipeline and compare the dashboards.
- D. Rebuild the entire gold set from scratch every release.

### 8. The chapter-02 attribution decision tree resolves a triad regression to one of four outcomes. On Monday, the aggregate faithfulness on the multi-hop slice dropped 8 points. Retrieval-only metrics (recall@5, MRR) are flat; the generator replay on the frozen multi-hop context shows a 7-point faithfulness drop. Which outcome does the tree return?

- A. Retrieval-only — escalate to the `rag-engineer` peer.
- B. Generation-only — own the fix on this side; bisect the generator diff (prompt, model, tool config).
- C. Coupling — convene a joint review.
- D. Joint / coincident release — handle each side independently.

### 9. Chapter 03 names three signature slices a product-shaped RAG suite must include. Which triple matches the chapter — and which failure does each slice catch?

- A. Head queries / tail queries / synthetic queries — each catches a sampling bias.
- B. **Distractor documents** (catch a generator grounding against on-topic-but-irrelevant chunks), **out-of-scope queries** (catch false-positive answers where the corpus has no answer), **hallucination-tempting prompts** (catch leading / false-premise phrasings that pressure the generator into fabrication).
- C. Short queries / long queries / medium queries — each catches a tokeniser regression.
- D. Multi-turn / single-turn / voice-transcribed — each catches an interface regression.

### 10. Chapter 03 explains why the out-of-scope slice must be scored with a **refusal-quality** rubric rather than reported inside the faithfulness and answer-relevancy aggregates. Which reason matches?

- A. Out-of-scope items are too rare to affect any aggregate.
- B. Faithfulness on an out-of-scope reply is undefined (no relevant retrieved context for the reply to be grounded in), and answer relevancy on a correct refusal is *supposed* to be low by design — including them in the triad aggregate penalises correct behaviour and rewards wrong behaviour.
- C. RAGAS raises an error when scoring out-of-scope items.
- D. Out-of-scope queries always come from synthetic data and therefore bias every metric upward.

### 11. Chapter 03 warns against 100%-synthetic (LLM-generated) gold sets even though they are easy to produce. What is the chapter's specific objection?

- A. Synthetic queries are more expensive than real traffic.
- B. LLM-generated queries phrase questions the way the LLM would phrase them, which is not how users phrase them; the eval improves on that phrasing and systematically drifts off production reality, so regressions only show up on real traffic.
- C. Synthetic queries always violate the PII policy.
- D. The RAGAS framework rejects synthetic items by construction.

### 12. Chapter 04 adds **noise sensitivity** as a diagnostic alongside the triad. Which statement is the chapter's reason for keeping noise sensitivity around even when faithfulness is flat?

- A. Noise sensitivity is cheaper to compute than faithfulness.
- B. Faithfulness can be fine (the reply is entailed by *some* chunk) while noise sensitivity is bad (the entailment is against an *irrelevant* chunk); noise-sensitivity regressions often precede faithfulness regressions, giving an early-warning signal for the "grounded but wrong chunk" pathology.
- C. Noise sensitivity replaces faithfulness entirely on online traffic.
- D. Noise sensitivity measures the user's typing noise, not retrieval noise.

### 13. Chapter 04 gives three distinct interpretations of combinations of context utilisation and faithfulness. Which pairing is wrong?

- A. **Low utilisation + high faithfulness** → context-window bloat; the retriever is surfacing chunks the generator isn't using. Cost signal to the `rag-engineer` peer: tighter top-k or better reranker.
- B. **High utilisation + low faithfulness** → the generator is misusing retrieved text (paraphrasing / mis-quoting). Rubric-anchor or generator-side fix.
- C. **Utilisation trending toward 0 on a slice** → retrieval is surfacing on-topic-but-not-load-bearing chunks; coordinate with `rag-engineer` on chunking / reranking.
- D. **Low utilisation + high faithfulness** → the generator is hallucinating; raise the faithfulness threshold immediately.

### 14. Chapter 04 presents a worked silent-regression catch: aggregate faithfulness stays flat at 0.92 all quarter, but noise sensitivity on the distractor-heavy slice climbs from 0.08 to 0.14 while faithfulness on the same slice slips 0.02 (within noise floor). Which reading matches the chapter?

- A. Noise sensitivity has drifted but the triad is green, so no action is warranted.
- B. The regression is real and is on the generator side (the generator is increasingly grounding against irrelevant chunks on the distractor slice); the diagnostic fires weeks before the triad aggregate would — fix with rubric-anchor tightening plus a generator prompt clarifier, and page on diagnostic-only regressions the same way you page on triad regressions.
- C. The regression is a retrieval issue; escalate directly without running the chapter-02 attribution tree.
- D. Noise sensitivity is noisy by default; raise the per-slice threshold until the alert stops firing.

### 15. Chapter 05 fixes the scope contract between this module and the `rag-engineer` peer. Which pairing matches the chapter?

- A. This module owns the retriever, reranker, chunker, and index. The peer owns metric definitions, rubrics, and paging policy.
- B. This module owns metric definitions, the labelled eval suite, the attribution machinery, the rubrics, the trace attribution (`app.rag.*`), the paging policy, and the release-gate eval criteria. The `rag-engineer` peer owns the retriever, reranker, chunker, index, corpus, retrieval-side prompt hooks (HyDE / self-query / multi-query), and the retrieval-side SLIs (latency, index freshness).
- C. Both roles own everything jointly; there is no boundary.
- D. The LLM-application engineer owns all of it; neither of the two roles above is involved.

### 16. Chapter 05 calls out two mirror anti-patterns — one for each side of the boundary. Which pair matches?

- A. Eval engineer *silently loosens a threshold or drops a slice from the aggregate* to make a regressed retriever look fine. `rag-engineer` peer *hand-tunes retriever behaviour to specific queries in the gold set* so the per-query scores improve without the retriever generalising. Both compromise the measurement.
- B. Eval engineer writes rubrics; `rag-engineer` tunes the retriever. There are no anti-patterns.
- C. Eval engineer owns the corpus; `rag-engineer` owns the rubrics.
- D. Both roles should fully share every artefact — there is no scope contract.

---

## Answer key

1. **B.** Chapter 01's table pairs each leg of the pipeline with one triad metric: retrieval precision (context precision), retrieval coverage (context recall), generation-groundedness-vs-retrieved-chunks (faithfulness), generation-vs-user-question (answer relevancy). Options A and D scramble those pairings.
2. **B.** Chapter 01 states that faithfulness is scored against the retrieved chunks and answer relevancy against the user question — neither requires a gold answer — which is "what makes them viable as online-eval SLIs… on sampled production traffic where no reference answer exists." Context precision (with reference) and context recall both benefit from a labelled gold set.
3. **B.** Chapter 01's "Faithfulness / Failure modes" section names this exact case as **paraphrase-vs-entailment slack**: a strict entailment judge marks unsupported, a lenient one marks supported. The chapter's fix is to pin the judge's entailment threshold through framework config or anchor language — `supported = a competent reader would accept the retrieved passage as the source`.
4. **A.** Chapter 01's framework-choice table: RAGAS ships all four metrics (reference implementation); TruLens's stock triad is groundedness + context relevance + answer relevance, and context recall is explicitly not shipped as a top-level TruLens metric (recall requires reference answers TruLens's runtime doesn't assume you have); DeepEval's contextual metrics are rubric-driven and pytest-native.
5. **A.** Chapter 02's "Why the triad alone doesn't attribute" section lays out two example regressions — a retrieval change and a generator change — that produce the same set of triad-metric movements. The triad tells you "something moved"; it does not tell you "what." That is the motivation for the retrieval-only metrics and the two replay patterns in the rest of the chapter.
6. **C.** Chapter 02's "Retrieval-only metrics" section names recall@k, MRR, and hit rate as the core triple, with precision@k and nDCG as additional ranking-aware variants. All four are computed from the retrieved chunk-id list against a labelled gold set with no judge call, which is what makes them cheap and CI-friendly.
7. **B.** Chapter 02's "Fixed-context replay" section defines the pattern exactly this way: freeze a small subset of the gold set with the retrieval verbatim, replay through the new generator, score with the triad's *generation* metrics (faithfulness / answer relevancy), skip context precision and recall (constant by construction). Any delta on the replay is generation-only.
8. **B.** The scenario is the chapter-02 worked attribution. Retrieval-only metrics flat ⇒ retrieval is not the cause on this slice; generator replay on the frozen context dropped ⇒ the regression is on the generator. The outcome the tree returns is **generation-only**, and the chapter's next step is to bisect the generator diff (prompt template, model snapshot, tool config).
9. **B.** Chapter 03 names the three signature slices in exactly these terms: distractor documents (catching a lazy generator grounding against on-topic-but-wrong chunks), out-of-scope queries (catching false-positive answers), and hallucination-tempting prompts (leading / false-premise phrasings that pressure the generator into fabrication).
10. **B.** Chapter 03 is explicit: faithfulness on an out-of-scope reply is undefined (no relevant retrieved context), answer relevancy on a correct refusal is *supposed* to be low by design, so including these items in those aggregates penalises correct behaviour. The chapter's fix is a dedicated **refusal-quality** rubric (`correct_refusal | false_answer | false_refusal | correct_answer`) applied to the out-of-scope slice only.
11. **B.** Chapter 03's "harvesting real traffic responsibly" section names this precise failure mode: LLM-generated queries phrase things the way the LLM would phrase them, the eval improves on that phrasing, and the metrics "drift systematically off production reality" — regressions appear only on real traffic. Paraphrase-augmentation on a thin real-query slice is OK; a 100%-synthetic suite is not.
12. **B.** Chapter 04's "Noise sensitivity" section makes this exact argument: faithfulness can pass because the reply is entailed by *some* chunk, while the entailment is against an *irrelevant* one — noise sensitivity resolves the "grounded but wrong" pathology directly, and chapter 04 notes that noise-sensitivity regressions often *precede* faithfulness regressions, which is why it is kept as an early-warning diagnostic.
13. **D.** Options A, B, and C are the chapter-04 diagnostic combinations verbatim. Option D is the reverse: chapter 04 does not conclude "the generator is hallucinating" from low utilisation + high faithfulness — it concludes the retriever is over-fetching and the signal is a *cost / grounding-fit* diagnostic to coordinate with the `rag-engineer` peer, not a reason to raise the faithfulness threshold.
14. **B.** Chapter 04's worked silent-regression catch describes exactly this scenario. The regression is real, on the generator side, detected weeks before the triad aggregate would move, and the chapter's rule is that a *diagnostic-only* regression is still a regression — wire each diagnostic to the same alert channel as the triad, with the same threshold-and-holdover pattern (mod-107).
15. **B.** Chapter 05's "What this module owns" and "What the `rag-engineer` peer owns" sections match this pairing. The eval engineer owns metrics, suite, attribution, rubrics, trace attribution, paging, and release-gate criteria; the peer owns retriever, reranker, chunker, index, corpus, retrieval-side prompt hooks (HyDE / self-query / multi-query), and retrieval SLIs.
16. **A.** Chapter 05's "Regressions that span the boundary" and "The gold set is fixed for a release" sections name these two mirror anti-patterns. Eval engineer silently loosening a threshold or dropping a slice to avoid a regression is "how the eval becomes theatre"; `rag-engineer` hand-tuning retriever behaviour to specific gold-set queries makes the gold set stale against new production traffic. Both compromise the eval as a defensible measurement.
