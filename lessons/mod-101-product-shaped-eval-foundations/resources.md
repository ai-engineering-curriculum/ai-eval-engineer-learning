# Resources for mod-101-product-shaped-eval-foundations

Primary references cited across the chapters and exercises. Where a resource is paywalled (ISO), the entry says so.

## Standards and frameworks

- **NIST AI Risk Management Framework (AI 100-1, 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. Four functions (Govern, Map, Measure, Manage); this track's plans are Measure-function artefacts.
- **NIST AI RMF Generative AI Profile (AI 600-1, 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Risk list and suggested actions per RMF function, specific to generative AI.
- **ISO/IEC 25059:2023 — Quality model for AI systems.** <https://www.iso.org/standard/80655.html>. Extends ISO/IEC 25010's product quality model with AI-specific characteristics. Paywalled.
- **ISO/IEC 25010:2011 — Systems and software Quality Requirements and Evaluation (SQuaRE) — System and software quality models.** <https://www.iso.org/standard/35733.html>. The base quality model that 25059 extends. Paywalled; public introductions widely available.
- **ISO/IEC 42001:2023 — AI management systems.** <https://www.iso.org/standard/81230.html>. Referenced for the peer Governance-family track's scope; not implemented in this module.
- **EU Artificial Intelligence Act (Regulation (EU) 2024/1689).** <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. Referenced for the peer Governance-family track's scope.
- **OWASP Top 10 for Large Language Model Applications.** <https://owasp.org/www-project-top-10-for-large-language-model-applications/>. Cross-referenced from `mod-108`.
- **MITRE ATLAS.** <https://atlas.mitre.org/>. Adversary tactics against AI systems; cross-referenced from `mod-108`.
- **Google SAIF (Secure AI Framework).** <https://safety.google/cybersecurity-advancements/saif/>. Cross-referenced from `mod-108` and `mod-112`.

## Reliability and SLOs

- **Google SRE Workbook, chapter 2: "Implementing SLOs."** <https://sre.google/workbook/implementing-slos/>. Vocabulary (SLI / SLO / SLA) used verbatim across this module.
- **Google SRE Book, chapter 4: "Service Level Objectives."** <https://sre.google/sre-book/service-level-objectives/>. Background reading.

## LLM-as-judge and benchmark methodology (reference; depth in other modules or peer tracks)

- **Zheng et al. 2023 — Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena.** <https://arxiv.org/abs/2306.05685>. Judge bias controls (position, self-preference) referenced in chapter 05 and deepened in `mod-104`.
- **Liu et al. 2023 — G-Eval: NLG evaluation using GPT-4 with better human alignment.** <https://arxiv.org/abs/2303.16634>. Rubric-driven judging referenced in chapter 02 and deepened in `mod-104`.
- **Kim et al. 2024 — Prometheus 2: An open-source LM specialized in evaluating other LMs.** <https://arxiv.org/abs/2405.01535>. Open judge model referenced in `mod-104`.
- **Es et al. 2023 — RAGAS: Automated Evaluation of Retrieval-Augmented Generation.** <https://arxiv.org/abs/2309.15217>. RAG-triad; deepened in `mod-105`.

## Public benchmarks (as shape templates)

- **Hendrycks et al. 2020 — Measuring Massive Multitask Language Understanding (MMLU).** <https://arxiv.org/abs/2009.03300>.
- **Cobbe et al. 2021 — Training Verifiers to Solve Math Word Problems (GSM8K).** <https://arxiv.org/abs/2110.14168>.
- **Jimenez et al. 2023 — SWE-bench: Can Language Models Resolve Real-World GitHub Issues?** <https://arxiv.org/abs/2310.06770>. Plus **SWE-bench Verified** (OpenAI, 2024): <https://openai.com/index/introducing-swe-bench-verified/>.
- **Yao et al. 2024 — τ-bench: A benchmark for tool-agent-user interaction.** <https://arxiv.org/abs/2406.12045>.
- **Zhou et al. 2023 — WebArena: A realistic web environment for building autonomous agents.** <https://arxiv.org/abs/2307.13854>.
- **Mialon et al. 2023 — GAIA: A benchmark for general AI assistants.** <https://arxiv.org/abs/2311.12983>.
- **Berkeley Function-Calling Leaderboard (BFCL).** <https://gorilla.cs.berkeley.edu/leaderboard.html>.
- **Qin et al. 2023 — ToolLLM: Facilitating LLMs to master 16000+ real-world APIs (ToolBench).** <https://arxiv.org/abs/2307.16789>.
- **Liu et al. 2023 — AgentBench: Evaluating LLMs as agents.** <https://arxiv.org/abs/2308.03688>.

## Application-layer eval frameworks and observability (referenced across the track)

- **OpenTelemetry Gen-AI semantic conventions.** <https://opentelemetry.io/docs/specs/semconv/gen-ai/>.
- **OpenInference span attributes.** <https://github.com/Arize-ai/openinference>.
- **Arize Phoenix.** <https://docs.arize.com/phoenix>.
- **Langfuse.** <https://langfuse.com/docs>.
- **W&B Weave.** <https://weave-docs.wandb.ai/>.
- **Braintrust.** <https://www.braintrust.dev/docs/start>.
- **LangSmith.** <https://docs.smith.langchain.com/>.
- **Promptfoo.** <https://www.promptfoo.dev/docs/intro/>.
- **DeepEval.** <https://docs.confident-ai.com/>.
- **TruLens.** <https://www.trulens.org/>.
- **UK AISI Inspect agent harness.** <https://inspect.aisi.org.uk/>.

## Model cards and reporting

- **Mitchell et al. 2018 — Model Cards for Model Reporting.** <https://arxiv.org/abs/1810.03993>. Referenced in exercise-03 stretch goal.

## Peer curriculum interfaces

- **`model-evaluation-engineer-learning`** — <https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning> — statistical methodology, benchmark construction, calibration methodology.
- **`ai-evaluation-engineer-learning` (Governance family, level 25)** — <https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning> — release assurance, audit trail, regulator interface.
- **`ai-risk-engineer-learning`** — <https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning> — harm modelling, red-team dataset generation.
- **`agentic-ai-engineer-learning`** — <https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning> — agent implementation (systems this track evaluates in `mod-103`).
- **`rag-engineer-learning`** — <https://github.com/ml-engineering-curriculum/rag-engineer-learning> — RAG engineering (systems this track evaluates in `mod-105`).
- **`llm-application-developer-learning`** — <https://github.com/ai-engineering-curriculum/llm-application-developer-learning> — LLM app engineering (systems this track evaluates).

<!-- needs-research: on the next research cycle, add live-in-window job-posting evidence for each of the six criteria families and for each of the eight benchmark failure modes covered in chapter 05; refresh links if any of the public benchmark URLs above have moved. -->
