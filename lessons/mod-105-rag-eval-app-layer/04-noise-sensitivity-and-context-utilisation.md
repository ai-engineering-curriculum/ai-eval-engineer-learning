# Noise Sensitivity and Context Utilisation: Diagnostics That Keep Regressions from Going Silent

## Motivation

Faithfulness is flat. Answer relevancy is flat. Context precision is flat. Aggregate looks good. Six weeks later a class of user complaints surfaces on social — a well-known failure mode has been quietly regressing on a slice the top-line SLIs don't cover. This is the *silent regression* problem, and the fix is not "add more slices" (chapter 03 already argued for slices) — it is a specific set of diagnostic measurements that sit *next to* the triad and quantify behaviours the triad's aggregates cannot see.

Two behaviours matter most for RAG:

- **Noise sensitivity** — how much does the reply degrade when retrieval surfaces a plausibly-related-but-irrelevant chunk. A generator that is robust to noise scores similarly with and without the irrelevant chunk; a generator that grounds against noise scores much worse when the noise is injected. RAGAS ships this as a first-class metric (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/noise_sensitivity/>).
- **Context utilisation** — how much of the retrieved context did the reply actually use. A reply that answers correctly but only referenced 5% of the retrieved context has spare capacity a competitor won't; a reply that "uses" 95% of context but with unsupported paraphrase is bloating for the judge. RAGAS's `ContextUtilization` measures this shape (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/#context-utilization>).

Both are cheap diagnostics with judge calls under the hood; both attach to the mod-102 trace the same way the triad does; and both are what a program owner (mod-112) points to when asked *"how do we know this isn't quietly getting worse."* This chapter walks the two, adds two more diagnostics that fit the same slot (answer completeness and refusal-quality), and finishes with the pattern for wiring them so a regression on a diagnostic pages before a user does.

## Core concepts

### Noise sensitivity: the metric

**Definition.** RAGAS's `NoiseSensitivity` measures the fraction of *incorrect* claims in a reply that were caused by irrelevant retrieved chunks. It requires:

- The query
- The reference answer (from the gold set)
- The retrieved context
- The generated reply

The metric decomposes the reply into claims, marks each claim as correct or incorrect (against the reference answer), and among the incorrect claims, checks which ones are traceable to irrelevant retrieval — i.e., the incorrect claim is entailed by a chunk that isn't a relevant chunk. The ratio *is* the noise sensitivity: 0 means every incorrect claim came from somewhere other than retrieved noise; higher means noise is driving errors.

The metric has a `focus` parameter — `relevant` (default) counts noise-driven errors on relevant retrieval; `irrelevant` counts noise-driven errors on irrelevant retrieval; the total noise sensitivity is the sum. See the RAGAS docs above.

**Why you want it as a diagnostic.**

- Faithfulness alone can be fine (the reply is entailed by *some* chunk) while noise sensitivity is bad (the entailment is against an irrelevant chunk). A generator that grounds fluently against the wrong chunk passes faithfulness and fails noise sensitivity.
- Noise sensitivity resolves the "grounded but wrong" pathology directly. Chapter 01's distractor-heavy slice was designed to make this measurable; noise sensitivity is the metric that reads that slice.
- Regressions in noise sensitivity often *precede* regressions in faithfulness — the generator starts leaning on irrelevant chunks first, then the reader-visible faithfulness drops as the leaning becomes more common. Trending noise sensitivity gives an early warning.

**How to run it.** RAGAS's implementation ingests a `Dataset` shape with the four fields above. Score on the distractor-heavy slice from chapter 03 and on the overall gold set separately — the delta between the two is the surface's sensitivity to distractors specifically.

**Failure modes to design around.**

- **Requires a reference answer.** Not viable on unlabelled online traffic. Fix: run offline on the gold set and on the harvested labelled subset of production traffic; complement with an online-only diagnostic (see "context utilisation" below).
- **Claim decomposition variance (same as faithfulness).** Fix: pin the decomposer, log claims per span, treat variance as noise floor.
- **Noise sensitivity of *correct* replies is 0 by construction.** The metric only counts incorrect claims. A generator that gets everything right on the gold set will show 0 noise sensitivity — that isn't a signal of robustness, only of correctness on this slice. Balance with a **distractor-injection stress test**: take a subset of gold items, deliberately add a plausible-looking distractor to the retrieved context, and re-score the reply. A robust generator's reply barely changes; a fragile generator adopts the distractor.

### Context utilisation: the metric

**Definition.** How much of the retrieved context contributed to the reply. RAGAS implements it as `ContextUtilization` — a per-chunk judge call that asks whether the chunk was "used" in the reply, aggregated to a per-query score. TruLens's `ContextRelevance` is adjacent but distinct: TruLens measures per-chunk *relevance to the question*, not per-chunk *use in the reply*.

**Why you want it as a diagnostic.**

- **Low utilisation with high faithfulness = context window bloat.** The retriever is surfacing chunks the generator isn't using; the generator is (correctly) grounding against the small useful subset. Cost regression: mod-111. Signal to the `rag-engineer` peer to consider a tighter top-k or a better reranker (chapter 05).
- **High utilisation with low faithfulness = the generator is *misusing* context.** The generator is leaning on retrieved text but paraphrasing / mis-quoting it. Rubric anchor problem (mod-104/chapter-01) or generator model bug.
- **Utilisation trending toward 0 on a slice = retrieval is surfacing on-topic-but-not-load-bearing chunks.** Chunking or reranking regression (chapter 05).

**How to run it.** RAGAS's implementation reads the retrieved context list and the reply from the trace; no reference answer required. This makes it a *viable online-eval diagnostic* — you can compute it on sampled production traffic.

**Failure modes to design around.**

- **"Used" is a soft judgement.** A chunk that supplies one background sentence and a chunk that supplies the whole answer both count. Fix: emit per-chunk verdicts on the span (`app.rag.chunk_verdicts`) so a reviewer sees which chunks the judge counted as used, and *weight* by chunk length or by fraction-of-chunk-used if your surface's answer distribution makes the flat count misleading.
- **Judge over-credits chunks the reply happens to touch on.** A reply that mentions a section number that a chunk contains scores that chunk as used, even if the reply's actual claim came from somewhere else. Fix: anchor the judge prompt on entailment ("was the reply's claim derivable from this chunk") rather than on surface overlap.
- **Undefined on refusals.** A correct refusal doesn't "use" any chunk. Fix: return `not_applicable` on refusals; exclude from aggregate.

### Answer completeness: the third diagnostic

**Definition.** How much of the *reference answer's* content is present in the reply. RAGAS ships this as `ResponseCompleteness` (called `AnswerCorrectness`'s recall side in some framework versions) — a claim-decomposition of the reference answer, checking each claim against the reply. Distinct from `AnswerRelevancy` (which asks about alignment with the question, not coverage of the reference answer).

**Why you want it as a diagnostic.**

- Answer relevancy is high on a reply that addresses the question. It is also high on a reply that addresses *part* of the question well and skips the rest. Answer completeness is what distinguishes those. A drop in completeness with flat relevancy is the classic "the reply looks fine, but it's only answering half the question" failure.
- On multi-hop / multi-part queries (chapter 03's multi-chunk slice), completeness is the metric that catches the case where the retriever surfaced both required chunks but the generator only used one.

**How to run it.** Requires a reference answer. Same tooling as faithfulness / context recall — RAGAS's claim-decomposition machinery works across all three.

**Failure modes.**

- **Requires a reference answer.** Offline only.
- **Over-penalises terser correct replies.** If the reference answer includes background context and the reply skips it correctly, completeness drops. Fix: mark reference answer claims as *load-bearing* vs *ancillary* and score completeness on load-bearing only.

### Refusal quality: the fourth diagnostic

**Definition.** For the out-of-scope slice from chapter 03, how correctly did the reply refuse. Not RAGAS-native; author a rubric per mod-104/chapter-01. Two dimensions:

- **Refusal precision**: given the reply refused, was it *right* to refuse (query truly out of scope). Complement of the false-refusal rate.
- **Refusal recall**: given the query was out of scope, did the reply refuse. Complement of the false-answer rate.

**Why you want it as a diagnostic.** Faithfulness and answer relevancy are undefined or misleading on out-of-scope items (chapter 01). Without a refusal-quality diagnostic, the whole out-of-scope slice sits outside the metric aggregates and its regressions don't page anybody. A single rubric that returns `{correct_refusal, false_answer, false_refusal, correct_answer}` on out-of-scope items (labelled from chapter 03) is enough.

**How to run it.** A DeepEval / RAGAS / TruLens judge with an absolute rubric per mod-104/chapter-01, scored on the out-of-scope slice only. mod-108 goes deeper on this same rubric shape for the safety family.

**Failure modes.**

- **Rubric drifts to punish hedging.** "I don't have this in the docs, but generally X" — is that a refusal or a false answer? Rubric anchors must call the shot.
- **Slice contamination.** An out-of-scope item gets re-classified in-scope after a corpus update but not re-labelled. Refusal-quality drops for a reason unrelated to the generator. Fix: chapter 03's re-audit discipline.

### How the diagnostics compose with the triad

The four diagnostics and the four triad metrics slot together in one table:

| Metric | Purpose | Requires reference answer | Viable online | Slice it reads |
|---|---|---|---|---|
| Faithfulness | Generation, groundedness | No | Yes | All (except refusals) |
| Answer relevancy | Generation, question-alignment | No | Yes | All (except refusals) |
| Context precision | Retrieval, chunk relevance | Not with `ContextRelevance`; yes with `LLMContextPrecision` | Partially | All |
| Context recall | Retrieval, chunk coverage | Yes | No | Labelled gold set |
| **Noise sensitivity** | Diagnostic, grounding-against-noise | Yes | No | Distractor-heavy + gold |
| **Context utilisation** | Diagnostic, cost / grounding-fit | No | Yes | All (except refusals) |
| **Answer completeness** | Diagnostic, multi-part-answer coverage | Yes | No | Multi-hop + gold |
| **Refusal quality** | Diagnostic, out-of-scope handling | No | Yes on labelled OOS | Out-of-scope |

Reporting shape: each metric is a per-slice mean with denominators. The dashboard has a row per metric and a column per slice. Regressions on any cell page the owning team; regressions on any *diagnostic* cell page even when the aggregate triad is green.

### The diagnostic dashboard: what to alert on

- **Noise sensitivity trending up** on the distractor-heavy slice — precedes faithfulness regression; page before the user does.
- **Context utilisation trending down** with flat top-k — the retriever is padding; escalate to `rag-engineer` on cost grounds even if the triad is flat.
- **Context utilisation trending up** with dropping faithfulness — the generator is misusing more of the context; escalate to the generator side.
- **Answer completeness drop on multi-hop slice** with flat answer relevancy — the generator is answering the easier half; escalate to prompt / model.
- **Refusal recall drop on out-of-scope slice** with flat everything else — false-answer regression on out-of-scope; escalate immediately (this is the "quiet hallucination" pathway).

The rule is that a diagnostic-only regression is *still* a regression. If the alerting policy only fires on the triad, the diagnostic column is decoration. Wire each diagnostic to the same alert channel as the triad; use the same threshold-and-holdover pattern (mod-107).

### A worked silent-regression catch

Concrete scenario. Aggregate faithfulness is flat at 0.92 all quarter. Noise sensitivity on the distractor-heavy slice was 0.08 at quarter-start, is now 0.14. Faithfulness on the distractor-heavy slice is 0.89 (down 0.02, within noise-floor).

Interpretation: the generator is increasingly grounding against irrelevant chunks in the distractor slice. Faithfulness on the slice is not yet visibly moving because the judge decomposition is coarse and the incorrect-claim rate is low, but the *causal shape* is present. The regression is real, is on the generator side (rerun the chapter-02 attribution tree to confirm), and would have shown up on faithfulness aggregate in two-to-four weeks.

Without noise sensitivity in the suite, the regression is invisible until the user complaints arrive. With it, an alert fires when noise sensitivity crosses a slice-specific threshold, and the fix — rubric anchor tightening plus a generator prompt clarifier — ships four weeks earlier.

### Attributes on the mod-102 trace for the diagnostics

- `app.rag.noise_sensitivity` — the noise-sensitivity score on this reply (populated on gold-set items only, since it needs the reference answer).
- `app.rag.context_utilisation` — the context-utilisation score, populated on any item with retrieval.
- `app.rag.completeness` — the answer-completeness score, populated on gold-set items.
- `app.rag.refusal_quality` — one of `correct_refusal | false_answer | false_refusal | correct_answer`, populated on out-of-scope-labelled items.
- `app.rag.chunk_verdicts` — per-chunk usage / relevance verdicts, populated by both context precision and context utilisation for auditability.

### What this chapter does not cover

- The retriever tuning that reduces the distractor rate upstream. Chapter 05 handoff.
- The rubric anchor changes that shift what the noise-sensitivity claim-decomposer catches. mod-104/chapter-01.
- The CI-gate thresholds these diagnostics feed. mod-106.
- The paging policy for diagnostic-only regressions. mod-107.

## Summary

- Silent RAG regressions happen when the triad aggregates are green but a failure mode is quietly getting worse on a specific slice. Diagnostic metrics prevent this.
- **Noise sensitivity** measures how much irrelevant retrieval drives errors — early-warning signal for the "grounded but wrong chunk" pathology; runs on the labelled gold set / distractor-heavy slice.
- **Context utilisation** measures how much of the retrieved context the reply actually used — a joint cost and grounding-fit diagnostic; runs online.
- **Answer completeness** catches the "half-answered multi-part question" failure that answer relevancy misses; runs on the gold set.
- **Refusal quality** catches the false-answer-on-out-of-scope pathway; runs on the labelled out-of-scope slice.
- Compose the diagnostics next to the triad in a per-slice × per-metric dashboard; alert on diagnostic-only regressions the same way you alert on triad regressions.
- Attribute the extra `app.rag.*` diagnostic attributes onto the mod-102 trace so the diagnostic runs share the trace store with the triad.

Chapter 05 walks the scope contract with the `rag-engineer` peer track — which of the retriever / reranker / chunker / prompt fixes this module owns and which ones you hand off with a slice-labelled regression report.
