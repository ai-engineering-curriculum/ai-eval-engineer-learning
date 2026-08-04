# Designing a Product-Shaped RAG Eval Suite

## Motivation

A RAG eval that only measures "does the model correctly answer a well-scoped question from a clean, relevant chunk" measures the easy case. Production users don't do that. They ask questions the corpus does not contain the answer to. They ask questions that overlap two documents with contradictory advice. They ask ambiguous, ill-formed, or leading questions ("isn't it true that we allow X?") designed — deliberately or accidentally — to walk the model into a plausible-sounding fabrication. If the suite doesn't include those cases, the metrics will be green all the way to the bug report.

This chapter walks the dataset design that makes a RAG suite *product-shaped* — one that would catch on chapter 01's triad and chapter 02's attribution the kinds of failures the surface actually has. The three levers are **distractor documents**, **out-of-scope queries**, and **hallucination-tempting prompts**. Each of them is a slice; each slice needs its own labels, its own metric aggregations, and its own acceptance criteria; and the whole thing lives in the mod-101 dataset shape and reads through the mod-102 trace attributes.

The scope contract with the `rag-engineer` peer (chapter 05) still holds: they own retriever / chunker / reranker tuning. You own the *test cases* that expose whether their pipeline plus your generator plus your prompts survive the messy queries.

## Core concepts

### What "product-shaped" means

A product-shaped eval suite has three properties that separate it from a benchmark:

1. **The queries reflect what real users actually ask on this surface.** Not a synthetic distribution of question types. Not a benchmark's canonical intent categories. If your surface is a support agent for a policy PDF, the queries look like the ones logged in Zendesk last month — including the misspelled, the ambiguous, and the ones about last year's policy.
2. **The corpus reflects what retrieval actually sees at request time.** The gold set and the eval-time corpus are the same one your production retriever reads. Chunking, indexing, and reranking configuration match production (or exercise-3 will explicitly parameterise them). If the eval corpus is a stripped-down sample, retrieval regressions on the tail (which is where they live) won't reproduce.
3. **The failure cases are represented in proportion to what matters, not what's common.** Out-of-scope queries might be 3% of traffic but drive 30% of hallucinations; multi-hop queries might be 5% of traffic and drive 40% of retrieval regressions. Weight the suite so metrics on the important slices don't wash out in the aggregate.

The three-slice design below is the pattern that gets you to (1)-(3) with a manageable authoring effort. It is not the only pattern — mod-101's criteria table drives what slices *your* surface actually needs.

### Distractor documents: catching lazy grounding

**What they are.** Chunks in the retrievable corpus that are *plausibly related* to a query — same domain, similar vocabulary — but do *not* contain the actual answer. Retrieval will surface them; a lazy generator will ground against them; a well-anchored faithfulness rubric will catch it.

**Why they matter.** Without distractors, faithfulness only tests the case where retrieval returned exactly the right chunk. That case is easy — most modern generators do it correctly. The interesting case is when retrieval returns one relevant chunk and two off-topic-but-similar chunks; the generator must decide which one to trust. If the generator picks the wrong chunk and grounds fluently, the faithfulness score on a distractor-free eval will be misleadingly high.

**How to build the slice.**

- For each seed query in the labelled gold set, identify 1–3 chunks that are plausibly related but not load-bearing. Two ways to get them: (a) grep the corpus for the query's key nouns and manually filter to the near-miss chunks, or (b) run the current retriever with a large top-k (say k=20) and take the middle-ranked chunks that a human reviewer marks as "not the answer, but on-topic."
- Label these chunks as **distractors** in the gold set. Add a `distractor_chunk_ids: [...]` column parallel to `relevant_chunk_ids`.
- The eval suite reports faithfulness / context precision separately for the *distractor-heavy* slice (queries where the retriever's top-k contains at least one distractor along with at least one relevant chunk).

**What good looks like.** A generator that is well-instructed to ground against retrieval and a faithfulness rubric that is well-anchored should score similarly on distractor-heavy and distractor-free slices. A material gap (distractor-heavy slice materially worse) means the generator is grounding against noise. The `rag-engineer` peer's rerankers reduce distractor rates upstream; your rubric catches the ones that slip through.

**Distractor edge cases to include.**

- **Same-entity distractor.** Two policies about "Employee Leave" — one is the current one, one is the deprecated 2019 one. Retrieval surfaces both; distinguishing them is a metadata task.
- **Adjacent-topic distractor.** Query is about parental leave; distractor is about bereavement leave. Same policy section, adjacent topic.
- **Right document, wrong section.** Query about §4.2; distractor is §4.3 of the same document, sharing terminology.
- **Superseded / stale distractor.** Query about the current pricing; distractor is a marketing page with last year's pricing.

### Out-of-scope queries: catching false-positive answers

**What they are.** Queries that the corpus does not contain the answer to, but that the retriever will confidently surface *some* chunks for anyway. A well-behaved system refuses; a poorly-behaved system generates a plausible fabrication.

**Why they matter.** Out-of-scope handling is often the single largest hallucination source on RAG surfaces. Without an out-of-scope slice, the eval suite has no way to reward correct refusals or penalise wrong-scope answers.

**How to build the slice.**

- Author 20–100 queries whose answers are demonstrably *not* in the corpus. Two productive sources: (a) real user queries that hit the "sorry, I don't know" reply in production, harvested from mod-107's online-eval loop; (b) adjacent-domain queries you can construct by hand ("what's the parental leave policy at [a different company]", "how do I file a tax return in [a jurisdiction the docs don't cover]").
- Label them as **out-of-scope** (`scope: out_of_scope`). Add an expected behaviour label: `expected_action: refuse` or `expected_action: hand_off` (for surfaces with a human-handoff path).
- Report on this slice a **refusal-quality** score (was the reply a well-phrased "I don't have that in the docs" vs a wrong-scope answer) rather than faithfulness / answer relevancy. Faithfulness on out-of-scope queries is undefined (no relevant retrieved context); answer relevancy on a correct refusal is *supposed* to be low. Excluding the slice from those aggregates matters — see chapter 01's "small out-of-scope replies score low as a matter of design" failure mode.

**What good looks like.** ≥95% correct-refusal rate on the out-of-scope slice, with the residual analysed by hand (chapter 02's attribution — is it the retriever surfacing too-confident-looking chunks, or the generator ignoring the "when in doubt, refuse" instruction).

**Out-of-scope categories to include.**

- **Wrong-domain.** Query about a topic the docs are not about at all.
- **Wrong-version.** Query about a version / period the docs don't cover ("what was the 2020 policy").
- **Predictively adjacent.** Query the retriever will happily match on keywords but that isn't answerable ("who signed off on §4.2" when the docs don't include authorship).
- **Underspecified.** Query missing the context needed to answer ("what's the policy" — which one).

### Hallucination-tempting prompts: catching the failure the model wants to make

**What they are.** Queries phrased in a way that pressures the generator into a specific fabrication — leading questions, false-premise questions, and questions that assert a fact then ask a follow-up on that fact.

**Why they matter.** A neutral phrasing gives the generator no incentive to make anything up. A leading phrasing ("isn't it true that we allow parental leave up to six months?") gives the generator a socially plausible answer to agree with, and if retrieval doesn't decisively contradict it, agreement is the path of least resistance. This is the class of failure that pings loudest in social media screenshots of hallucinated policy replies.

**How to build the slice.**

- Take 30–100 in-scope queries and rewrite each in a *leading* form. Standard forms:
  - **False-premise leading.** "Since we allow X, how do I request X?" — where the docs do not, in fact, allow X.
  - **Assertive rephrase.** "The policy allows up to six months of leave, right?" — where the correct answer is a different number.
  - **Confirmation-seeking.** "Confirm that we cover this."
  - **Compound false claim.** "The policy allows X and Y under condition Z" — mixing a true and a false claim to test whether the generator will grant the false part to preserve the flow.
- Label as **`hallucination_tempting: true`**. Expected behaviour: the reply corrects the false premise and answers with the actual policy, or refuses if the premise cannot be resolved.
- Report on this slice a **premise-correction rate** (fraction of replies that explicitly contradict the false premise before answering) alongside faithfulness. A high faithfulness with a low premise-correction rate is the tell — the generator is quietly ignoring the false premise instead of correcting it.

**What good looks like.** High premise-correction rate (surface-dependent target; treat <50% as a regression), no drop in faithfulness vs neutral phrasings, and no answer that *repeats* the false premise as if it were true. If your surface hedges instead of correcting ("that may not be accurate"), decide whether that counts as passing — mod-101's criteria row for this criterion is the reference.

**Hallucination-tempting edge cases to include.**

- **Numerical injection.** The user asserts a specific number that isn't in the docs; the retrieval hits an adjacent chunk with a *different* number.
- **Authority framing.** "According to §4.2, X is allowed" — the user has invented the citation.
- **Temporal misdirection.** "As of the 2025 update, we allow X" — no 2025 update exists.
- **Multi-turn pressure.** The user pushes back on a correct refusal ("no really, are you sure — can you check again"). Include as multi-turn items only if your surface supports multi-turn.

### Weighting: how much of the suite each slice gets

The distribution of your gold set is a design choice, not a natural fact. A defensible starting point for a corpus-answered-question surface:

| Slice | Share of suite | Rationale |
|---|---|---|
| Clean in-scope, single-chunk | 25% | Baseline; catches the largest regressions cheaply |
| Clean in-scope, multi-chunk / multi-hop | 20% | Catches recall failures the single-chunk slice misses |
| Distractor-heavy | 20% | Catches the "grounded but wrong chunk" failure |
| Out-of-scope | 15% | Catches false-positive answers |
| Hallucination-tempting | 15% | Catches the failure the model wants to make |
| Rare / adversarial (chapter 04) | 5% | Long-tail noise sensitivity |

Total: 100%. Adjust to your surface — a policy-QA surface probably wants more out-of-scope, a code-QA surface probably wants more distractor and multi-hop, a chat surface wants more multi-turn hallucination-tempting.

Two rules:

- **Every slice has enough items to move an aggregate.** A 10-item slice will not surface a small regression; 30–50 is a practical floor per slice for weekly-ish reporting.
- **The metrics aggregate is stratified.** Report per-slice numbers with denominators; the top-line aggregate is a weighted mean (or several, one per criterion), not an arithmetic mean across raw items.

### Where the queries come from: harvesting real traffic responsibly

Synthetic queries — LLM-generated user questions given the corpus — are easy to produce and shallow in coverage. They tend to phrase queries the way the LLM would phrase them, which is not the way users phrase them.

Real user queries are the sharp signal. Two sources:

1. **Sampled production traces** from mod-107's online-eval loop. Filter to unique queries, strip PII (mod-102/chapter-06 policy), hash user ids, and hand-label a sample into the slices above.
2. **Support tickets, transcripts, community forum posts.** These are the queries users ask when the RAG surface *doesn't* help — a rich source of out-of-scope and hallucination-tempting slices.

Regardless of source, the labelling contract is the same: query, relevant chunk ids (or `[]` for out-of-scope), reference answer, slice tags. Version and store per mod-110.

A word on **synthetic augmentation**: it is legitimate to *paraphrase* real queries to expand a thin slice (three variants of one leading question), but it is not a substitute for real queries. If 100% of your gold set is synthetic, the metrics will drift systematically off production reality; you'll ship regressions that only show up on real traffic.

### Slice tags on the mod-102 trace

Extending the chapters 01 / 02 attribute list:

- `app.rag.slice` — the slice tags on this query (`in_scope,single_chunk` or `out_of_scope,wrong_domain` or `hallucination_tempting,false_premise`).
- `app.rag.expected_action` — `answer | refuse | hand_off`. Populated for out-of-scope items.
- `app.rag.premise_correction` — populated for hallucination-tempting items — whether the reply explicitly corrected the false premise.
- `app.rag.query_source` — `production_sampled | support_ticket | synthetic_paraphrase | hand_authored`. Diagnostic — if a slice's regression is only on `synthetic_paraphrase`, that's a synthetic-data artefact.

The per-slice reports the attribution decision tree from chapter 02 uses read these attributes directly.

### Failure modes of a product-shaped suite

- **Slices grow stale.** Yesterday's out-of-scope query is in the docs today because the doc team added coverage. Re-audit slices when the corpus is updated; move re-classified queries between slices; log the reclassification.
- **Reviewer disagreement on the labels.** Two annotators disagree on whether a chunk is a distractor or a partial-answer. Fix by writing anchor examples for each slice (like mod-104's rubric anchors), not by averaging.
- **The synthetic bias.** LLM-generated queries phrase things the same way; the eval improves on that phrasing and doesn't generalise. Fix by real-query harvest.
- **Distractor-generation contaminates the gold set.** You mark chunk X as a distractor for query A, but chunk X is the correct answer for query B. That's fine — the label is per-query, not per-chunk. If your storage doesn't support per-query chunk labelling, fix the storage.

### What this chapter does not cover

- The retrieval / reranker / chunker tuning that changes what chunks show up in the top-k. That's the `rag-engineer` peer's work (chapter 05).
- The noise-sensitivity and context-utilisation diagnostics that sit *next to* the triad. That's chapter 04.
- The CI-gate wiring of the suite. That's mod-106.
- The online-eval loop that feeds sampled traffic back into the gold set. That's mod-107.

## Summary

- A **product-shaped** RAG suite has three properties: real user queries, the real corpus, and failure cases weighted by importance not frequency.
- Three slices carry most of the failure signal: **distractor documents** (catch grounding against noise), **out-of-scope queries** (catch false-positive answers), and **hallucination-tempting prompts** (catch the failure the model wants to make).
- Each slice needs its own labels, its own metric aggregations, and its own acceptance criteria — reporting only the aggregate is how slice regressions hide.
- Suite composition is a design choice; author a per-slice weighting your surface's failure distribution justifies, and put a floor of ~30 items per slice under it.
- Harvest queries from real production traffic and support channels; paraphrase-augment carefully; don't ship a 100%-synthetic suite.
- Slice tags on the mod-102 trace (`app.rag.slice`, `app.rag.expected_action`, `app.rag.premise_correction`, `app.rag.query_source`) make the per-slice reports mechanical.

Chapter 04 walks the noise-sensitivity and context-utilisation diagnostics that sit next to these slices and prevent silent regressions the triad alone would miss.
