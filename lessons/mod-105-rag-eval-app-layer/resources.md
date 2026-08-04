# Resources for mod-105-rag-eval-app-layer (RAG Evaluation at the Application Layer)

Primary references cited across the chapters and exercises. Prefer these to blog posts — the papers, the framework docs, and the metric definitions move faster than the write-ups about them, and this track relies on the primary sources being read *from the URL* each release cycle.

## Foundational RAG-eval literature

- **Es et al. 2023 — "RAGAS: Automated Evaluation of Retrieval Augmented Generation."** <https://arxiv.org/abs/2309.15217>. The paper that introduces the reference-free RAG-triad measurement pattern (faithfulness, answer relevancy, context relevance). Companion library: <https://github.com/explodinggradients/ragas>. Cited across chapters 01, 02, and 04.
- **Saad-Falcon et al. 2024 — "ARES: An Automated Evaluation Framework for Retrieval-Augmented Generation Systems."** <https://arxiv.org/abs/2311.09476>. Complementary framework; trains lightweight judges on synthetic queries. Useful cross-reference on the design trade-offs.
- **Chen et al. 2023 — "Benchmarking Large Language Models in Retrieval-Augmented Generation."** <https://arxiv.org/abs/2309.01431>. Introduces four RAG capabilities — noise robustness, negative rejection, information integration, counterfactual robustness — that map cleanly to chapter 03's slice design and chapter 04's diagnostics.

## Judge-scored triad and diagnostics — framework references

- **RAGAS.** <https://docs.ragas.io/>. Reference implementation of the RAG triad and the diagnostics. Metrics used in this module:
  - Faithfulness — <https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/>
  - Answer relevancy — <https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/answer_relevance/>
  - Context precision (LLM-based and non-LLM) — <https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/>
  - Context recall (LLM-based and non-LLM) — <https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/>
  - Noise sensitivity — <https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/noise_sensitivity/>
  - Non-LLM context precision — <https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/non_llm_context_precision/>
  - Repo: <https://github.com/explodinggradients/ragas>.
- **TruLens.** <https://www.trulens.org/>. Feedback-function library that ships the RAG triad in a trace-native shape (groundedness, context relevance, answer relevance). Recall is not shipped; author it separately. RAG-triad concept doc: <https://www.trulens.org/getting_started/core_concepts/rag_triad/>. Repo: <https://github.com/truera/trulens>.
- **DeepEval.** <https://docs.confident-ai.com/>. pytest-native. Contextual metrics used in this module:
  - `FaithfulnessMetric` — <https://docs.confident-ai.com/docs/metrics-faithfulness>
  - `AnswerRelevancyMetric` — <https://docs.confident-ai.com/docs/metrics-answer-relevancy>
  - `ContextualPrecisionMetric` — <https://docs.confident-ai.com/docs/metrics-contextual-precision>
  - `ContextualRecallMetric` — <https://docs.confident-ai.com/docs/metrics-contextual-recall>
  - Repo: <https://github.com/confident-ai/deepeval>.

<!-- needs-research: on the next research cycle, re-verify the current class names for the RAGAS metrics (the library renames periodically), and confirm the DeepEval contextual metric class names and docs paths. Also check whether TruLens has added a top-level recall feedback function since the last cycle. -->

## Retrieval-only metrics and IR references (chapter 02)

- **TREC common evaluation measures — Voorhees & Harman 2005 primer.** <https://trec.nist.gov/pubs/trec21/appendices/measures.pdf>. The reference source for MRR, Recall@k, Precision@k, and nDCG definitions used in this module. Older papers, but the definitions are stable.
- **Manning, Raghavan, Schütze — *Introduction to Information Retrieval*.** <https://nlp.stanford.edu/IR-book/>. Free online; chapters 8 and 11 cover the retrieval metrics and how to interpret them.
- **`sklearn.metrics.ndcg_score`.** <https://scikit-learn.org/stable/modules/generated/sklearn.metrics.ndcg_score.html>. Reference implementation of nDCG.
- **BEIR benchmark.** <https://github.com/beir-cellar/beir>. Heterogeneous IR benchmark; the reference implementation of nDCG@k / Recall@k / MRR@k on retrieval tasks that this module's retrieval-only metrics parallel.
- **`ir_measures`.** <https://ir-measur.es/en/latest/>. Python wrapper over `pytrec_eval` and other implementations of TREC's canonical metrics; useful when you want the retrieval-only metrics without hand-rolling them.

## Distractor / noise / hallucination-tempting literature (chapters 03 and 04)

- **Shi et al. 2023 — "Large Language Models Can Be Easily Distracted by Irrelevant Context."** <https://arxiv.org/abs/2302.00093>. Establishes the "grounded but wrong chunk" failure mode chapter 03's distractor slice is designed to catch.
- **Liu et al. 2023 — "Lost in the Middle: How Language Models Use Long Contexts."** <https://arxiv.org/abs/2307.03172>. Position bias inside the retrieved context — the model attends preferentially to the top and bottom, so distractors in the middle behave differently from distractors at the edges. Relevant to chapter 03's distractor authoring.
- **Yoran et al. 2024 — "Making Retrieval-Augmented Language Models Robust to Irrelevant Context."** <https://arxiv.org/abs/2310.01558>. Training-time approaches to noise robustness; useful cross-reference for the `rag-engineer` peer on chapter 05.

## Judge-family and rubric references (inherited from mod-104)

The RAG triad is judge-scored. The mod-104 discipline on rubric anchors, judge biases, framework choice, calibration, and tier routing all apply here. Rather than duplicate mod-104's `resources.md`, cite it directly:

- **mod-104 `resources.md`** — <../mod-104-llm-as-judge-in-product/resources.md>. Primary references for Zheng et al. 2023 (LLM-as-judge biases), Liu et al. 2023 (G-Eval), Panickssery et al. 2024 (self-preference), Dubois et al. 2024 (length-controlled scoring), and the Prometheus 2 / JudgeLM open-source judge families this module's rubrics can run against.

## Trace and observability backends (chapters 02, 03, and 04 attribution attributes)

The RAG-triad and diagnostic scores attach back to the mod-102 trace as EVALUATOR-kind spans with `app.rag.*` attributes. The mod-102 resources file is authoritative on the backend list; this module reuses it:

- **mod-102 `resources.md`** — <../mod-102-trace-instrumentation/resources.md>. Primary references for Arize Phoenix, Langfuse, W&B Weave, Braintrust, LangSmith, OpenInference, and OpenTelemetry GenAI semantic conventions.

## Peer-track curriculum interface (chapter 05)

- **`rag-engineer` role curriculum.** The peer track that owns retriever / reranker / chunker / index / corpus tuning. Verify the exact roles-registry path in this org; the scope contract in chapter 05 cites it as the owner for anything upstream of the retrieved-context boundary. <!-- needs-research: on the next research cycle, add the direct link to the `rag-engineer` role's curriculum in the org's roles registry, plus a link to its retrieval-SLI documentation if published. -->

## Governance and safety cross-references

- **NIST AI Risk Management Framework (AI 100-1, 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. The Measure function is where the RAG-triad and its diagnostics land. The per-slice × per-metric dashboard from exercise-04 is a Measure-function artefact.
- **NIST AI RMF Generative AI Profile (AI 600-1, 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Suggested actions for GAI risks — the confabulation risk maps directly onto the hallucination-tempting slice in chapter 03 and the noise sensitivity diagnostic in chapter 04.
- **OWASP Top 10 for Large Language Model Applications.** <https://owasp.org/www-project-top-10-for-large-language-model-applications/>. LLM01 (Prompt Injection) and LLM09 (Overreliance) overlap the hallucination-tempting and out-of-scope slices; the safety-family deep dive is in mod-108.

## Retrieval-adjacent stacks worth knowing (for the peer boundary)

Not part of this module's ownership per chapter 05, but useful references when the peer sends a retriever change notice or a chunker diff:

- **LlamaIndex.** <https://docs.llamaindex.ai/>. Common orchestration layer for RAG; the metric-evaluation section of the docs cross-references RAGAS and TruLens.
- **LangChain retrievers.** <https://python.langchain.com/docs/concepts/retrievers/>. Retriever abstraction the peer often builds against.
- **Haystack.** <https://docs.haystack.deepset.ai/docs/evaluation>. Alternative RAG orchestration stack with a native evaluation module.

<!-- needs-research: on the next research cycle, re-verify current URLs for RAGAS, TruLens, DeepEval, LlamaIndex, LangChain, and Haystack; check whether ARES has a maintained companion library; and check whether the OWASP LLM Top 10 has released a new version (LLM01…LLM10 mapping may have shifted). -->
