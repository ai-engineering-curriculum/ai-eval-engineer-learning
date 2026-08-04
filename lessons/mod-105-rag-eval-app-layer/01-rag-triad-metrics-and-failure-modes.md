# The RAG Triad: Faithfulness, Answer Relevancy, Context Precision and Recall

## Motivation

A retrieval-augmented generation surface has three coupled failure modes and they are easy to confuse for each other. The reply looks great, but is grounded in the wrong document. The reply is grounded in the right document, but ignores the user's question. The reply answers the question and is faithful, but retrieval never surfaced the passage that actually matters. Each of these looks like the same complaint on a dashboard — *"the answer is wrong"* — and each has a different fix.

The **RAG triad** — faithfulness, answer relevancy, and the context precision / recall pair — is the metric decomposition that turns *"the answer is wrong"* into *"which of the three pieces of the pipeline broke"*. It is named and popularised by TruLens (<https://www.trulens.org/>) and RAGAS (<https://docs.ragas.io/>); DeepEval ships equivalent metrics with slightly different implementations. This chapter walks the four metrics — what each is trying to measure, how the mainstream frameworks compute it, and the failure modes each one has as an LLM-judge-scored SLI.

The bias controls from mod-104's chapter 02, the anchor discipline from mod-104's chapter 01, and the calibration discipline from mod-104's chapter 04 all still apply here. What the RAG triad adds is a *decomposition* — three or four measurements per request that separate retrieval from generation and separate generation-faithfulness from generation-relevancy. Skip the decomposition and every regression is a whodunnit.

## Core concepts

### The three pieces of the pipeline the triad measures

A RAG request has three legs, each of which can independently fail:

1. **Query → retrieved context.** The retriever's job. Passes the top-k chunks (with their scores) to the generator.
2. **Retrieved context → reply, staying faithful.** The generator's job, part one. Reply's factual claims are supported by the retrieved context.
3. **User's question → reply, actually answering it.** The generator's job, part two. Reply addresses what the user asked, not something the retrieved documents happen to say.

Each leg gets a triad metric:

| Metric | Which leg it measures | Where the ground truth (if any) comes from |
|---|---|---|
| **Context precision** | Retrieval, quality dimension: are the retrieved chunks relevant to the question | Reference contexts (gold) or a judge |
| **Context recall** | Retrieval, coverage dimension: did retrieval surface every chunk needed to answer | Reference contexts (gold) or a judge |
| **Faithfulness** | Generation → retrieved context: are reply claims supported by the retrieved chunks | The retrieved chunks themselves (judge-scored, no gold answer required) |
| **Answer relevancy** | Generation → user question: does the reply actually address the question | The user question (judge-scored, no gold answer required) |

Two properties of this table matter:

- The **generation metrics don't require a gold answer.** Faithfulness is scored against retrieval; answer relevancy is scored against the question. This is what makes them viable as online-eval SLIs — you can compute them on sampled production traffic where no reference answer exists.
- The **retrieval metrics do require a labelled gold set** to be sharp — reference contexts (RAGAS) or reference answers a judge can back-derive contexts from. Chapter 02 handles the retrieval-only metrics (recall@k, MRR, hit rate) that don't need a judge at all.

### Faithfulness (a.k.a. groundedness)

**Definition.** The fraction of factual claims in the reply that are supported by (entailed by) the retrieved context. RAGAS decomposes the reply into a list of claims, checks each against the retrieved chunks, and returns the ratio of supported claims to total claims (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/>). TruLens's `Groundedness` feedback does effectively the same at a coarser granularity (<https://www.trulens.org/getting_started/core_concepts/rag_triad/>). DeepEval's `FaithfulnessMetric` is judge-scored on the reply and the retrieved context (<https://docs.confident-ai.com/docs/metrics-faithfulness>).

**Why you want it as an SLI.** A frontier model asked to summarise a retrieved policy PDF will happily add a plausible clause the PDF does not contain — the hallucination is grammatical and matches the surrounding tone, and no reader will catch it. Faithfulness gives you a per-request signal that this happened. On a support-agent surface, it is often the single most defensible quality SLI you have — the user's question and the reply's helpfulness are subjective, but "the claim is or is not in the retrieved chunk" is close to a fact.

**Failure modes to design around.**

- **Claim-decomposition failure.** The reply is one sentence with three implicit claims and the judge decomposes it into one claim, or into six. Score varies with the decomposition, not with reality. RAGAS's decomposition step is itself a judge call — its output is stochastic. Fix: log the decomposed claim list on the EVALUATOR span (`app.rag.claims`) so a reviewer can see what the judge counted; run the metric twice at different temperatures and treat the variance as noise floor.
- **Paraphrase gets scored as unsupported.** The reply says *"the policy allows this"* and the retrieved chunk says *"§4.2 permits this action"*. A strict entailment judge marks unsupported; a lenient one marks supported. Fix: pin the judge's *entailment threshold* — either through the framework's config or through anchor language ("supported = a competent reader would accept the retrieved passage as the source"). See mod-104/chapter-01.
- **Judge over-credits the reply's authoritative voice.** A confident reply gets more benefit-of-the-doubt than a hedged one. Fix: mod-104/chapter-02 length and style controls — score against retrieval, not against the reply's phrasing.
- **Faithfulness of `not_applicable` cases.** For refusals, greetings, and out-of-scope replies (chapter 03), the reply contains no factual claims and the metric is undefined. Fix: return the `not_applicable` label rather than a numeric 1.0; RAGAS returns NaN in this case — chapter 03 covers handling.

### Answer relevancy

**Definition.** How well the reply actually addresses the user's question, independent of whether the reply is grounded or correct. RAGAS's `AnswerRelevancy` reverses the question — it asks a judge to generate the question the reply is best answering, then computes an embedding similarity between the generated question and the original (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/answer_relevance/>). DeepEval's `AnswerRelevancyMetric` uses a judge directly against the question and reply (<https://docs.confident-ai.com/docs/metrics-answer-relevancy>). TruLens exposes `Relevance` feedback with either implementation as a plug-in.

**Why you want it as an SLI.** A grounded reply that answers the wrong question is a bad reply. This happens most when the retriever surfaces an on-topic but off-question chunk — the model answers what the chunk is *about* rather than what the user *asked*. Answer relevancy separates that failure mode from a faithfulness failure.

**Failure modes to design around.**

- **The reverse-question trick misfires on multi-part questions.** A user asks A and B; the reply addresses only A. RAGAS's reverse-question judge, given the partial reply, generates a question that matches A alone; embedding similarity to the original A+B is still high; the metric misses the missing half. Fix: for multi-part surfaces, score answer relevancy per intent (chapter 03's intent decomposition) rather than per user turn.
- **Embedding similarity is not question-alignment.** Two questions can have high embedding similarity ("How do I reset my password?" vs "Where is password reset in settings?") without one answering the other. Fix: use a direct judge-scored implementation (DeepEval, or RAGAS's `NoiseSensitivity`) when you care about the answer-alignment axis and the reverse-question heuristic is fooling you.
- **Verbose non-answers score higher than terse answers.** The reply pads with tangential material that keeps the reverse-question judge's generated question close to the original; the actual answer never appears. Fix: mod-104/chapter-02 length control on the judge prompt, and a coupled diagnostic (chapter 04's context utilisation) — an answer with high answer-relevancy but low context utilisation is the flag.
- **Small out-of-scope replies score low as a matter of design.** A correct refusal to an out-of-scope question ("we don't have information on that in our docs") is *supposed* to have low answer-relevancy to the question. Fix: mark out-of-scope items in the dataset (chapter 03) and exclude them from the answer-relevancy aggregate, or route them through a separate refusal-quality rubric.

### Context precision

**Definition.** Of the chunks retrieval surfaced, what fraction are relevant to the question — and (in RAGAS's implementation) is the ordering right. RAGAS's `LLMContextPrecision` walks the top-k retrieved chunks and, for each one, asks a judge whether it is relevant to the question given the reference answer; it then applies an ordering-aware average (higher-ranked relevant chunks weighted more) (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/>). TruLens's `ContextRelevance` is per-chunk relevance without the ordering penalty; DeepEval's `ContextualPrecisionMetric` is similar to RAGAS's with a slightly different aggregate.

**Why you want it as an SLI.** Low context precision is what a stakeholder sees when the retriever surfaces off-topic chunks — the model then either grounds against irrelevant material (faithfulness looks fine to the judge but the reply is wrong) or answers around the chunks (answer relevancy suffers). Context precision is the *upstream* quality signal that predicts both.

**Failure modes to design around.**

- **Requires a reference answer or reference contexts.** RAGAS's `LLMContextPrecision` needs one; without it, you fall back to `ContextRelevance` (a per-chunk judge call against the question only). Fix: for online-eval on unlabelled traffic, use `ContextRelevance` or per-chunk answer-relevancy; save `LLMContextPrecision` for the offline gold-set runs.
- **Ordering weight is opinionated.** The RAGAS aggregate weights top-ranked chunks more; if your surface concatenates retrieval into one long prompt where order doesn't matter to the generator, the ordering penalty is measuring the wrong thing. Fix: switch to the unordered mean or to per-chunk relevance if your generator does not care about chunk order.
- **Chunk boundaries confuse the judge.** A relevant paragraph split across two chunks gets marked as two half-relevant chunks. Fix: coordinate with the `rag-engineer` peer on chunking (chapter 05) — this is a chunking issue, not a metric issue, and forcing a re-tune of the metric to compensate hides the real problem.

### Context recall

**Definition.** Of the reference contexts required to correctly answer the question, what fraction did retrieval surface. RAGAS's `LLMContextRecall` decomposes the reference answer into claims and checks each claim against the retrieved chunks — the fraction of claims supported by *what was retrieved* is the recall score (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/>). DeepEval's `ContextualRecallMetric` is a similar decomposition. TruLens does not ship recall as a top-level triad metric — the RAG-triad in TruLens is context relevance + groundedness + answer relevance — because recall requires reference answers TruLens's runtime doesn't assume you have.

**Why you want it as an SLI.** Faithfulness and answer relevancy can both be high on a reply that missed the important information entirely — the model correctly answered a narrower question using the (partial) retrieval it got. Context recall is the metric that catches "retrieval didn't surface the passage that mattered." It is what distinguishes a *retrieval regression* from a *generation regression* on your gold set.

**Failure modes to design around.**

- **Requires a labelled gold set.** Not viable as an online-eval SLI on unlabelled production traffic. Fix: run it on the mod-101 offline gold set and, if you have the labels, on the mod-107 harvested gold-set subset. Chapter 02 covers the retrieval-only recall@k that is viable on the gold set without a judge.
- **Claim decomposition is stochastic (same as faithfulness).** Same fix — pin the decomposer, log the claim list, treat variance as noise floor.
- **Recall of a *poorly labelled* gold set is meaningless.** If the gold answer references a chunk that isn't actually load-bearing, recall drops for a reason unrelated to retrieval quality. Fix: audit the gold labels; when in doubt, prefer *"which chunks does a human need to answer this question"* to *"which chunks does the current reply happen to cite."*

### Framework choice for the triad

The three-way framework choice mirrors mod-104's chapter-03 discussion; the details for RAG:

| Framework | Faithfulness | Answer relevancy | Context precision | Context recall | Notes |
|---|---|---|---|---|---|
| **RAGAS** | Yes, claim-decomposed | Yes, reverse-question | Yes, ordering-aware | Yes, claim-decomposed | The reference implementation of the four metrics; least opinionated on trace shape; requires reference answers for precision/recall |
| **TruLens** | Yes, `Groundedness` feedback | Yes, `Relevance` | Yes, `ContextRelevance` (per-chunk, no ordering) | Not shipped as a top-level triad metric | The triad it names is groundedness + context relevance + answer relevance; recall is DIY |
| **DeepEval** | Yes, `FaithfulnessMetric` | Yes, `AnswerRelevancyMetric` | Yes, `ContextualPrecisionMetric` | Yes, `ContextualRecallMetric` | pytest-native; ships an assortment of contextual metrics with slightly different implementations from RAGAS |

Rules of thumb:

- If you already have judgement anchors and rubrics from mod-104 and want the metric to be *your rubric*, prefer DeepEval — its metrics accept a rubric-like `criteria` string and act as G-Eval judges.
- If you want the reference triad implementations to compare your surface against published RAGAS papers/blog posts, use RAGAS directly.
- If your team already runs on TruLens for feedback-function tracing, use TruLens for groundedness / relevance / context-relevance and pair with a RAGAS / DeepEval recall implementation.

Whatever framework, follow the mod-104 rules: pin the judge model, pin the judge prompt, log the rubric hash on the EVALUATOR span, and don't accept judges you have not calibrated (chapter 04 of mod-104).

### How the triad attaches to the mod-102 trace

Each triad metric emits an EVALUATOR-kind span attached to the LLM reply span (for faithfulness and answer relevancy) or to the RETRIEVER span (for context precision and recall). The mod-102/chapter-04 span-attribute schema gives the base attributes; the RAG-triad-specific extensions are:

- `app.rag.metric` — `faithfulness | answer_relevancy | context_precision | context_recall`
- `app.rag.score` — the numeric score, plus the `not_applicable` sentinel where applicable
- `app.rag.claims` — the decomposed claim list (for faithfulness and context recall) so the score is auditable
- `app.rag.chunk_verdicts` — the per-chunk relevance verdicts (for context precision) so a reviewer can see which chunk the judge dropped
- `app.rag.judge_prompt_hash` — the mod-104 rubric hash for the judge that produced the score

Log the metric on the same trace as the request. Chapter 02 covers how to *split* a regression across the metrics using these attributes; chapter 04 covers the diagnostic metrics that sit *next to* the triad to catch silent failures.

## Summary

- The RAG triad — **faithfulness, answer relevancy, context precision, context recall** — decomposes *"the answer is wrong"* into which leg of the pipeline broke.
- Faithfulness and answer relevancy are viable on unlabelled online traffic; context precision and recall benefit from a labelled gold set (RAGAS decomposes the reference answer into claims for both).
- Each metric has documented failure modes: claim-decomposition noise, paraphrase-vs-entailment slack, multi-part-question misses, ordering-weight opinionation, gold-label quality. Design around each rather than accepting a raw framework default.
- Frameworks differ: RAGAS is the reference implementation, TruLens is trace-native but skips recall, DeepEval is rubric-driven and pytest-native. Pick one per surface and stick with it, or run two in parallel and calibrate the delta (exercise-01).
- Attach every metric back to the mod-102 trace with the `app.rag.*` attributes so a regression is inspectable, not just a moving number on a dashboard.

Chapter 02 walks how to separate a retrieval regression from a generation regression using this decomposition plus the retrieval-only metrics — recall@k, MRR, hit rate on the labelled gold set.
