# Resources for mod-104-llm-as-judge-in-product (LLM-as-Judge in Product Pipelines: Rubrics, Bias Controls, and Judge-Tier Routing)

Primary references cited across the chapters and exercises. Prefer these to blog posts — the papers, the framework docs, and the model repositories move faster than the write-ups about them, and this track relies on the primary sources being read *from the URL* each release cycle.

## Foundational LLM-as-judge literature

- **Zheng et al. 2023 — "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena."** <https://arxiv.org/abs/2306.05685>. Establishes position bias, verbosity bias, and self-enhancement bias as systematic failure modes of LLM judges; introduces swap-and-average as the standard position-bias mitigation. Cited across chapters 01, 02, and 04.
- **Liu et al. 2023 — "G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment."** <https://arxiv.org/abs/2303.16634>. The chain-of-thought + form-filling + probability-weighted-decoding pattern that DeepEval / Braintrust / Promptfoo operationalise. Cited from chapter 03.
- **Panickssery, Bowman, Feng 2024 — "LLM Evaluators Recognize and Favor Their Own Generations."** <https://arxiv.org/abs/2404.13076>. Empirical evidence for self-preference / self-enhancement across frontier judges; motivates chapter 02's cross-family diagnostic.
- **Dubois et al. 2024 — "Length-Controlled AlpacaEval."** <https://arxiv.org/abs/2404.04475>. Length-controlled win-rate methodology — the reference for chapter 02's length-adjusted scoring.
- **LMSYS Chatbot Arena.** <https://lmsys.org/blog/2023-05-03-arena/>. Pairwise-preference arena; canonical example of pairwise rubric use in production. The Chatbot Arena leaderboard has evolved substantially — cross-check current methodology at <https://lmarena.ai/> before citing specific numbers.

## Open-source dedicated judge models

- **Kim et al. 2024 — "Prometheus 2: An Open Source Language Model Specialized in Evaluating Other Language Models."** <https://arxiv.org/abs/2405.01535>. Weights and inference recipe: <https://github.com/prometheus-eval/prometheus-eval>. 7B and 8×7B open-source evaluator LLMs; ingests a custom rubric at inference time.
- **Zhu et al. 2023 — "JudgeLM: Fine-tuned Large Language Models are Scalable Judges."** <https://arxiv.org/abs/2310.17631>. Code and weights: <https://github.com/baaivision/JudgeLM>. Pairwise-focused, with position and length bias mitigations trained into the model.

<!-- needs-research: on the next research cycle, check for newer open-source judge families (successors to Prometheus 2 and JudgeLM, or new entrants) with public weights and inference recipes suitable for self-hosting. Update chapters 03 and 05 accordingly. -->

## Judge frameworks (chapters 03 and 05)

- **DeepEval.** <https://docs.confident-ai.com/>. Python-first, pytest-native. The `GEval` metric is the primary G-Eval implementation; the framework also ships pre-built faithfulness, answer relevancy, contextual precision, and hallucination metrics. Repo: <https://github.com/confident-ai/deepeval>.
- **Braintrust — `autoevals`.** <https://www.braintrust.dev/docs>. Eval-first data model (experiments containing scored traces). The open-source `autoevals` library is at <https://github.com/braintrustdata/autoevals> and ships `LLMClassifier`, `ClosedQA`, `Battle`, and other graders.
- **Promptfoo — `llm-rubric` and other model-graded assertions.** <https://www.promptfoo.dev/docs/configuration/expected-outputs/model-graded/>. YAML / CLI-first; oriented toward eval-in-CI. Repo: <https://github.com/promptfoo/promptfoo>.

## Model-serving stacks for self-hosted judges

- **vLLM.** <https://docs.vllm.ai/>. High-throughput OpenAI-compatible server for open-source LLMs; the default runtime for a self-hosted Prometheus 2 or JudgeLM in this module. Repo: <https://github.com/vllm-project/vllm>.
- **Text Generation Inference (TGI).** <https://huggingface.co/docs/text-generation-inference/>. Hugging Face's alternative serving stack; also OpenAI-compatible endpoints.

## Statistics primary references (chapter 04)

Use the peer's Model-Development-family curriculum for full-methodology depth; the pointers below are what a product-side engineer reads to compute the quick-calibration statistic honestly.

- **Cohen's κ — Wikipedia.** <https://en.wikipedia.org/wiki/Cohen%27s_kappa>. Standard inter-rater agreement statistic for nominal / categorical labels. Landis-Koch interpretation bands (0.61–0.80 "substantial", 0.81–1.0 "almost perfect") are heuristics from a 1977 paper (Landis & Koch, *Biometrics*, DOI 10.2307/2529310) — do not treat as hard thresholds.
- **Spearman's ρ — Wikipedia.** <https://en.wikipedia.org/wiki/Spearman%27s_rank_correlation_coefficient>. Non-parametric rank correlation for ordinal data; the default for scoring against ordinal-anchored rubrics.
- **Kendall's τ — Wikipedia.** <https://en.wikipedia.org/wiki/Kendall_rank_correlation_coefficient>. More conservative rank statistic; preferred for very small N.
- **`scipy.stats.spearmanr`.** <https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.spearmanr.html>. Reference implementation.
- **`scipy.stats.kendalltau`.** <https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.kendalltau.html>. Reference implementation.
- **`scipy.stats.bootstrap`.** <https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.bootstrap.html>. Non-parametric bootstrap for confidence intervals; supports `method="BCa"`.
- **`sklearn.metrics.cohen_kappa_score`.** <https://scikit-learn.org/stable/modules/generated/sklearn.metrics.cohen_kappa_score.html>. Reference implementation for two-rater κ; the `weights` argument supports linear / quadratic weighting for ordinal use.

## Trace and observability backends (referenced from chapters 02 and 05)

The judge pipeline emits EVALUATOR-kind spans back to whichever trace backend the module surface uses; the mod-102 resources file is authoritative on the backend list, and this module reuses it.

- **Arize Phoenix.** <https://docs.arize.com/phoenix>. Self-hostable OpenInference-native trace viewer + eval hooks. Repo: <https://github.com/Arize-ai/phoenix>.
- **Langfuse.** <https://langfuse.com/docs>. Scores and traces attach on sessions and users; SaaS and self-host.
- **W&B Weave.** <https://weave-docs.wandb.ai/>. Judge scores can live next to W&B experiment runs.
- **Braintrust.** <https://www.braintrust.dev/docs>. Eval-first — the primary artefact is the scored experiment.
- **LangSmith.** <https://docs.smith.langchain.com/>. LangChain / LangGraph-native; also accepts OTel OTLP.
- **OpenInference specification.** <https://github.com/Arize-ai/openinference>. Source of truth for the EVALUATOR span kind and its attribute vocabulary.
- **OpenTelemetry GenAI semantic conventions.** <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Referenced for the `gen_ai.*` mirror attributes the EVALUATOR span carries alongside the OpenInference ones.

## RAG-eval framework references (bridge into mod-105)

- **RAGAS.** <https://docs.ragas.io/>. Faithfulness / answer relevancy / context precision / context recall as judge-scored metrics. Deepened in mod-105; this module's rubric + bias-control patterns apply directly.
- **TruLens.** <https://www.trulens.org/>. Feedback functions attaching judge scores to traces / spans; alternative to RAGAS at a similar level of the stack.

## Governance and safety cross-references

- **NIST AI Risk Management Framework (AI 100-1, 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. The Measure function is where judge scoring lands; the calibration document from chapter 04 is a Measure-function artefact.
- **NIST AI RMF Generative AI Profile (AI 600-1, 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Suggested actions for GAI risks map to judge-scoring choices for safety criteria (deepened in mod-108).
- **OWASP Top 10 for Large Language Model Applications.** <https://owasp.org/www-project-top-10-for-large-language-model-applications/>. The safety-family rubrics this module's pipeline scores against overlap with the LLM01 / LLM06 / LLM09 concerns.
- **ISO/IEC 25059:2023 — Quality model for AI systems.** <https://www.iso.org/standard/80655.html>. The quality characteristics that per-surface criteria (and their judge rubrics) implement. Paywalled.

## Peer curriculum interfaces

- **Model-Development-family `model-evaluation-engineer` (level 30).** The escalation target for chapter 04's full-methodology calibration work — variance decomposition, statistical power analysis, unequal-interval ordinal analysis, non-standard resampling. Curriculum lives in the Model-Development family's role catalogue.
- **Governance-family `ai-evaluation-engineer` (level 25).** The sign-off owner for the release-gate architecture that mod-112 assembles from this module's routing config + drift monitor + calibration docs.
- **`agentic-ai-engineer-learning`** and **`llm-application-developer-learning`** — the roles that build the surfaces this module scores.

<!-- needs-research: on the next research cycle, re-verify current URLs and current release notes for DeepEval, Braintrust `autoevals`, and Promptfoo — the frameworks release frequently and rename APIs. Confirm the Prometheus-eval GitHub org is still current (`prometheus-eval/prometheus-eval`) and re-check the JudgeLM repo path. Also check whether the current-generation frontier chat APIs expose top-token log-probabilities in a stable form that G-Eval's weighted-score mode can use across all three frameworks. -->
