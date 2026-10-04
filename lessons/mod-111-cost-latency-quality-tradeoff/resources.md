# Resources for mod-111-cost-latency-quality-tradeoff (Cost / Latency / Quality Trade-off Evaluation)

Primary references cited across the chapters and exercises. Prefer the official docs and the original papers over blog posts — vendor pricing, caching semantics, and reasoning-tier accounting change on a monthly cadence and summaries drift fast. Read from the URL each time you plan to build against a specific behaviour.

## Multi-objective evaluation and Pareto decision-making

The chapter 02 report shape (Pareto frontier, dominance, pre-registered rule, tie-breakers) sits on a long literature in multi-criteria decision analysis and pre-registration.

- **Belton, V., Stewart, T. "Multiple Criteria Decision Analysis: An Integrated Approach." Springer, 2002.** The reference text for multi-criteria decision analysis; the chapter 02 vocabulary (dominance, Pareto frontier, weighted sums as an anti-pattern) derives from this literature.
- **Deb, K. "Multi-Objective Optimization Using Evolutionary Algorithms." Wiley, 2001.** Pareto-front formalism; small-multiples 2D projections of a 4-D frontier is a technique originating in this community.
- **Nosek, B., Ebersole, C., DeHaven, A., Mellor, D. "The preregistration revolution." *PNAS* 115 (11), 2018.** <https://www.pnas.org/doi/10.1073/pnas.1708274114>. The empirical case for pre-registration as the anti-motivated-reasoning discipline chapter 02 adopts.
- **OSF — Preregistration guide.** <https://help.osf.io/article/158-create-a-preregistration>. Social-science reference for pre-registration format; chapter 02's "written before the numbers were seen, signed by product + engineering" rule is a lighter-weight analogue.
- **Google — "The AI Testing Playbook: multi-objective evaluation."** <https://developers.google.com/machine-learning/testing-debugging/common/overview>. Vendor-neutral guidance; cross-reference for the chapter 02 four-column shape.

## Model routing patterns and the cohort-preservation contract

Chapter 03's router evaluation discipline — the cohort-preservation contract, the oracle-based attribution, the always-frontier fallback.

- **Chen, L., Zaharia, M., Zou, J. "FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance." *TMLR* 2024.** <https://arxiv.org/abs/2305.05176>. The reference paper on tier-based LLM routing (cascade); introduces the "query → score → tier" chain the exercise-02 policy composes.
- **Ding, D., Mallick, A., Wang, C., Sim, R., Mukherjee, S., et al. "Hybrid LLM: Cost-Efficient and Quality-Aware Query Routing." *ICLR* 2024.** <https://arxiv.org/abs/2404.14618>. Classifier-gated routing with explicit quality-cost trade-off; mirrors the exercise-02 "classifier[mid] / classifier[frontier] / fallback" chain.
- **Hu, Q., Yao, J., Yu, F., et al. "RouteLLM: Learning to Route LLMs with Preference Data." *ICML* 2024.** <https://arxiv.org/abs/2406.18665>. Preference-based routing; chapter 03's oracle-attribution is the measurement counterpart to the paper's training objective.
- **Shnitzer, T., Ou, A., Silva, M., Soule, K., Sun, Y., Solomon, J., et al. "Large Language Model Routing with Benchmark Datasets." arXiv, 2024.** <https://arxiv.org/abs/2309.15789>. Benchmark-based routing; useful lens on where routing eval overfits.
- **Martian — "LLM Router Benchmark" (public leaderboard).** <https://leaderboard.withmartian.com/>. Public router comparisons across vendors and tasks; useful as a sanity check for the exercise-02 traffic distribution.
- **LangChain — Routing documentation.** <https://python.langchain.com/docs/how_to/routing/>. Framework-level routing primitives the exercise-02 policy composition can reuse.
- **LlamaIndex — Query routing.** <https://docs.llamaindex.ai/en/stable/examples/query_engine/RouterQueryEngine/>. Alternate framework-level reference.
- **NIST — "AI Measurement and Evaluation for Fairness and Equity."** <https://www.nist.gov/itl/ai-risk-management-framework>. Cohort-preservation parallels the fairness-across-subgroups discipline in the AI RMF; useful context for defending the chapter 03 contract to governance stakeholders.

## Distillation and replacement regression

Chapter 04's paired-comparison methodology, teacher-agreement rate, and coverage-and-quality plot.

- **Hinton, G., Vinyals, O., Dean, J. "Distilling the Knowledge in a Neural Network." *NIPS Deep Learning Workshop*, 2014.** <https://arxiv.org/abs/1503.02531>. The reference distillation paper; "student learns the teacher's soft outputs" is the pattern chapter 04's coverage-and-quality plot measures.
- **Gu, Y., Dong, L., Wei, F., Huang, M. "MiniLLM: Knowledge Distillation of Large Language Models." *ICLR* 2024.** <https://arxiv.org/abs/2306.08543>. Distillation of generative LLMs; failure modes documented here (style-match before task-match) are the chapter 04 anti-pattern.
- **Agarwal, R., Vieillard, N., Zhou, Y., Ramos, S., Munos, R., et al. "On-Policy Distillation of Language Models: Learning from Self-Generated Mistakes." arXiv, 2023.** <https://arxiv.org/abs/2306.13649>. On-policy variant; useful if the student is trained with the exercise-03's incumbent as the teacher.
- **McNemar, Q. "Note on the sampling error of the difference between correlated proportions or percentages." *Psychometrika* 12 (2), 1947.** The paired-binary significance test chapter 04 cites. Readable summary: <https://en.wikipedia.org/wiki/McNemar%27s_test>.
- **Benjamini, Y., Hochberg, Y. "Controlling the false discovery rate: a practical and powerful approach to multiple testing." *J. R. Statist. Soc. B* 57 (1), 1995.** <https://www.jstor.org/stable/2346101>. The FDR-correction method chapter 04 requires across cohort tests.
- **Holm, S. "A Simple Sequentially Rejective Multiple Test Procedure." *Scand. J. Statist.* 6 (2), 1979.** The Holm-Bonferroni alternative for small cohort counts.
- **scipy.stats — paired-t and McNemar implementations.** <https://docs.scipy.org/doc/scipy/reference/stats.html>. Reference library for the exercise-03 `stats/intervals.py`.
- **statsmodels — multiple-testing correction.** <https://www.statsmodels.org/stable/stats.html#multiple-testing-correction>. Benjamini-Hochberg / Holm-Bonferroni implementations the exercise-03 `stats/fdr.py` imports.

## App-altitude latency, streaming, and the TTFT / TPOT vocabulary

Chapter 05's app-altitude latency metrics, OTel-GenAI span attributes, retry-latency contract.

- **OpenTelemetry — GenAI semantic conventions.** <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. The canonical attribute vocabulary for LLM spans; chapter 05's instrumentation validator reads against this.
- **OpenTelemetry — specification (traces + spans).** <https://opentelemetry.io/docs/specs/otel/>. The underlying spec the mod-102 instrumentation sits on.
- **OpenInference — semantic conventions.** <https://github.com/Arize-ai/openinference/tree/main/spec>. Alternate GenAI span conventions widely used by vendor runtimes.
- **MLCommons — MLPerf Inference Benchmark Suite.** <https://mlcommons.org/benchmarks/inference-datacenter/>. The model-altitude / fleet-altitude reconciliation benchmark chapter 05 reads *down* from (via the `ai-infra-performance` peer).
- **MLCommons — MLPerf Inference LLM workload results.** <https://mlcommons.org/benchmarks/inference-datacenter/>. Current serving benchmark results across hardware targets; useful when the peer re-benchmarks and the reconciliation footer updates.
- **Agrawal, A., Panwar, A., Mohan, J., et al. "SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills." arXiv, 2023.** <https://arxiv.org/abs/2308.16369>. Serving-side primer on prefill/decode phases; context for why TTFT (prefill-bound) and TPOT (decode-bound) are measured separately.
- **Google SRE Workbook — "Latency and Performance."** <https://sre.google/sre-book/latency/>. The p50 / p95 / p99 discipline chapter 05 inherits; useful on why `mean` is banned.
- **Gil Tene — "How NOT to Measure Latency." Strange Loop 2015.** <https://www.youtube.com/watch?v=lJ8ydIuPFeU>. The reference talk on coordinated omission and percentile reporting; chapter 05's "global vs. cohort-weighted vs. worst-cohort p95" is a direct response.
- **HdrHistogram — specification.** <https://github.com/HdrHistogram/HdrHistogram>. The percentile data structure the exercise-04 percentile code can use for accurate tail-latency reporting without pre-aggregation loss.
- **AWS Architecture Blog — "Exponential Backoff and Jitter."** <https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/>. The retry-policy reference chapter 05 points to for the retry-storm failure mode.

## Prompt caching, Batch APIs, and reasoning-token accounting

Chapter 06's cache / batch / thinking-token mechanics and the pricing-snapshot discipline.

- **Anthropic — Prompt caching documentation.** <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>. The reference vendor caching surface; the chapter 06 `cache-read` and `cache-write` classes map directly onto the Anthropic token-usage attributes.
- **Anthropic — Message Batches API.** <https://docs.anthropic.com/en/docs/build-with-claude/batch-processing>. The ~50 % batch-discount surface chapter 06 names. Price details on the pricing page.
- **Anthropic — Pricing.** <https://www.anthropic.com/pricing>. Current per-million-token prices for input / output / cache-read / cache-write / batch — the exercise-05 pricing snapshot's reference.
- **Anthropic — Extended thinking.** <https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking>. Reasoning-token behaviour and billing semantics for the thinking-token column.
- **OpenAI — Prompt caching documentation.** <https://platform.openai.com/docs/guides/prompt-caching>. Alternate vendor caching surface; token classes map similarly.
- **OpenAI — Batch API.** <https://platform.openai.com/docs/guides/batch>. The OpenAI 50 % batch discount; the exercise-05 batch-vs-sync code adapts per vendor.
- **OpenAI — Pricing.** <https://openai.com/api/pricing/>. Current per-model prices.
- **OpenAI — Reasoning tokens.** <https://platform.openai.com/docs/guides/reasoning>. The reasoning-token billing semantics for o-series models.
- **Google — Gemini context caching.** <https://ai.google.dev/gemini-api/docs/caching>. The third major vendor's caching surface.
- **Google — Vertex AI pricing.** <https://cloud.google.com/vertex-ai/generative-ai/pricing>. Current per-model prices.
- **AWS Bedrock — Prompt caching.** <https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html>. The Bedrock caching surface; useful when the app routes through Bedrock.
- **AWS Bedrock — Pricing.** <https://aws.amazon.com/bedrock/pricing/>. Current Bedrock per-model prices.
- **FinOps Foundation — AI/ML working group.** <https://www.finops.org/wg/aiml/>. The reference FinOps vocabulary for generative-AI cost governance; the exercise-05 cost-centre roster and invoice reconciliation sit under this frame.
- **FinOps Foundation — Framework.** <https://www.finops.org/framework/>. The parent framework; chargeback / showback / cost-centre terminology.

<!-- needs-research: on the next research cycle, re-verify Anthropic / OpenAI / Google / AWS Bedrock pricing URLs and refresh the illustrative per-million-token numbers in exercise-05's reference snapshot YAML. -->

## Public benchmarks the peer owns (cross-reference, chapter 07 delegation)

The chapter 07 escalation to `model-evaluation-engineer` reads the peer's benchmarks; the links below are the common targets. The app-altitude eval team *reads* these; it does not run them.

- **MMLU — Massive Multitask Language Understanding.** <https://github.com/hendrycks/test>. The reference broad-knowledge benchmark.
- **GPQA — Graduate-Level Google-Proof Q&A.** <https://github.com/idavidrein/gpqa>. Reasoning-heavy benchmark.
- **HumanEval — code-generation benchmark.** <https://github.com/openai/human-eval>. Pass@1 on code tasks.
- **MMMU — Massive Multi-discipline Multimodal Understanding.** <https://mmmu-benchmark.github.io/>. Multimodal reasoning.
- **LiveCodeBench — contamination-controlled code benchmark.** <https://livecodebench.github.io/>. Code benchmark with time-based splits; useful when peer freshness matters.
- **HELM — Holistic Evaluation of Language Models.** <https://crfm.stanford.edu/helm/>. Stanford's broad-coverage framework; peer-owned reference.
- **LMSYS Chatbot Arena.** <https://lmarena.ai/>. Public leaderboard of head-to-head preference; useful as a sanity check for a candidate before paired replacement eval.
- **BIG-bench.** <https://github.com/google/BIG-bench>. 200+-task reference.
- **UK AISI Inspect.** <https://inspect.aisi.org.uk/>. The agent / safety eval harness peer teams often standardise on; cross-reference from mod-103 and mod-108.

## Trace backends and eval platforms (cross-reference)

Chapter 02's lineage-key reads join against these backends; the exercise-04 and exercise-05 code reads from them.

- **Arize Phoenix — documentation.** <https://docs.arize.com/phoenix>. OSS trace backend + evaluator surface.
- **Langfuse — documentation.** <https://langfuse.com/docs>. OSS trace + evaluator platform (also hosted).
- **Braintrust — documentation.** <https://www.braintrust.dev/docs>. Hosted eval + prompt-management platform.
- **W&B Weave — documentation.** <https://weave-docs.wandb.ai/>. Weights & Biases's LLM evaluation surface.
- **LangSmith — observability.** <https://docs.smith.langchain.com/observability>. LangChain-native backend.
- **SigNoz — documentation.** <https://signoz.io/docs/>. OSS APM / tracing; the OTel-native alternative when the org already runs it.

## SRE, SLOs, and alerting for the trade-off report's rollback trigger

The rollback trigger is a mod-107 online-loop SLO; the alerting shape is Google SRE applied to the eval surface.

- **Google SRE Book — Service Level Objectives (chapter 4).** <https://sre.google/sre-book/service-level-objectives/>. The SLI / SLO / SLA vocabulary.
- **Google SRE Workbook — Implementing SLOs.** <https://sre.google/workbook/implementing-slos/>. The SLI-definition and error-budget shape the rollback trigger reads against.
- **Google SRE Workbook — Alerting on SLOs.** <https://sre.google/workbook/alerting-on-slos/>. Burn-rate alert thresholds the rollback trigger uses.
- **OpenSLO — specification.** <https://openslo.com/docs/overview>. OSS spec for declaring SLOs as code; the exercise-01 `rollback/trigger.yaml` can follow this shape.
- **Prometheus — alerting rules.** <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>. The alerting substrate the rollback trigger wires against.

## Governance and program-owner cross-references (deeper coverage in mod-112)

The trade-off decisions and their artefacts are evidence under multiple governance frames.

- **NIST AI Risk Management Framework (AI 100-1, 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. The Measure and Manage functions cover the chapter 02 report (Measure) and chapter 03 – 06 artefacts (Manage).
- **NIST AI RMF Generative AI Profile (AI 600-1, 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Continuous-monitoring suggested actions map directly to the mod-107 integration and the chapter 06 reconciliation.
- **ISO/IEC 42001:2023 — AI management systems.** <https://www.iso.org/standard/81230.html>. The AI management system standard; chapter 02's signed-report discipline is 42001-relevant. Paywalled.
- **EU Artificial Intelligence Act (Regulation (EU) 2024/1689).** <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. Post-market monitoring obligations for high-risk systems (Articles 72 – 73); the rollback-trigger discipline is one of the artefacts a high-risk deployment would produce.

## Peer curriculum interfaces

Chapter 07 draws the delegation boundary; these peers are the targets.

- **`model-evaluation-engineer-learning` (Model-Development family, level 30).** The peer that owns the offline benchmark suites (MMLU, GPQA, HumanEval, MMMU, LiveCodeBench) and the MLPerf-shaped serving benchmark. Chapter 07 defines the escalation contract.
- **`ai-infra-performance-learning` (Infra family).** The peer that owns the serving fleet — kernels, batching, KV-cache management, GPU-hours per token, capacity planning. Chapter 07 defines the escalation contract.
- **`llm-application-developer-learning`.** The product-team peer whose prompts and chains feed the trade-off report's `prompt_hash` and `chain_hash` lineage keys.
- **`agentic-ai-engineer-learning`.** The agent-authoring peer whose traces feed the chapter 02 cohort matrix via mod-102; cross-reference from mod-103.
- **`ai-evaluation-engineer-learning` (Governance family, level 25).** The governance peer that signs off on release gates and continuous-monitoring architecture; cross-reference from mod-112.

<!-- needs-research: on the next research cycle, confirm the arxiv URLs for FrugalGPT / Hybrid LLM / RouteLLM resolve and update to the published-venue DOI if available; re-verify MLCommons MLPerf Inference URL; refresh the vendor pricing / caching / batch URLs that drift most often. -->
