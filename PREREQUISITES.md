# Prerequisites for the AI Evaluation Engineer track

This curriculum starts at **level 30 (deep specialist, AI Engineering family — peer to `model-evaluation-engineer` on the ML Engineering ladder and to `senior-ml-engineer` at the same level)**. It does not re-teach build-altitude LLM application / RAG / agent development, and it does not re-teach model-eval methodology depth. Before working through it, you should already be comfortable with the items below. The [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) track (level 20) is the canonical place to acquire the ML foundation, [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning) / [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) / [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) the canonical places to acquire the AI application, RAG, and agent building depth, and [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) (level 10) below that for engineering craft.

## Required

- **Intermediate Python** — functions, classes, modules, virtual environments, packaging (`uv` / `poetry`), structured logging, unit testing, and the ability to read a moderately complex open-source library (you will read Phoenix / Langfuse / Weave / Braintrust / Promptfoo / DeepEval / RAGAS internals).
- **LLM application development fluency** — you have built and shipped at least one small LLM application: prompt design, tool / function calling, streaming, error handling, and at least one production-shape framework (LangChain, LangGraph, LlamaIndex, OpenAI Agents SDK, Anthropic Claude Agent SDK, raw provider SDK, or equivalent). This curriculum evaluates such systems; it does not teach how to build them.
- **RAG working familiarity** — you have wired an embedding model, a vector store, and a generation model into a working retrieval pipeline once. RAG *engineering* depth is not required — mod-105 owns RAG *eval*.
- **Agent working familiarity** — you have implemented at least one ReAct-shaped tool-calling agent end-to-end. Multi-agent orchestration is not required — mod-103 owns trajectory eval; agent implementation depth is owned by the peer `agentic-ai-engineer-learning`.
- **At least one LLM API used in anger** — OpenAI / Anthropic / Gemini / open-source LLM via an API, including function-calling. You will be evaluating these.
- **Classical ML evaluation fluency** — confusion-matrix-derived metrics (precision / recall / F1, accuracy), ROC / PR, train / dev / test discipline, at the level of `ml-engineer-learning`. Deep classical ML eval methodology is owned by the peer `model-evaluation-engineer-learning`.
- **Basic statistics** — distributions, expectations, the *meaning* of a confidence interval and a p-value, and the bootstrap intuition. mod-101 assumes this and links to `model-evaluation-engineer` for depth.
- **Observability basics** — you have used OpenTelemetry, Datadog, Prometheus, or an equivalent tracing stack once. mod-102 extends this to Gen-AI conventions.
- **Linux command line and Git** — assumed at the level of `ai-infra-junior-engineer-learning`.
- **SQL and a Pandas / Polars equivalent** — trace warehouses, eval-result stores, and cost / quality reports live here.
- **CI/CD working familiarity** — you have wired GitHub Actions (or GitLab CI / CircleCI / Buildkite) for at least one project. mod-106 wires evals into that.

## Recommended

- **One eval integration shipped at any depth** — added a Promptfoo YAML to a repo, wired a DeepEval pytest, ran a RAGAS notebook, or added a Weave / Langfuse trace hook to a small app. mod-102, mod-105, and mod-106 will land faster if you have.
- **One agent trajectory debugged in a real trace viewer** — spent an hour in a Phoenix / Langfuse / Weave trace tree finding a bad tool call. mod-103 will feel intuitive if you have.
- **One production incident postmortem written or read** — mod-107 (regression detection) and mod-112 (incident-driven eval investment) land more concretely for engineers who have felt an incident cycle.
- **Exposure to A/B testing** — a college course, a previous job, or the Kohavi-Tang-Xu chapters. mod-107 covers the *interface* to the A/B platform; deep methodology is owned by `model-evaluation-engineer` / `senior-ml-engineer`.
- **API and CLI credits** — a budget of $50-$100 in frontier-API credits for judge work across mod-104, mod-105, mod-107, and mod-108, plus $20-$40 in mid-tier or open-source model credits for judge-tier bake-offs.
- **Docker / container fluency** — mod-103 (sandboxed agent replay), mod-108 (guardrail sandbox), and mod-110 (multi-runner orchestration) will feel routine if you have.

## Out of scope as prerequisites

You do **not** need to know any of the following before starting; they are either taught in this curriculum or owned by a peer/upstream track:

- Arize Phoenix / Langfuse / W&B Weave / Braintrust / LangSmith / Promptfoo / DeepEval / RAGAS / TruLens / Humanloop / Galileo / Patronus — taught here.
- Inspect (UK AISI) agent harness for product-shaped tasks — taught here.
- LLM-as-judge rubric design and bias controls (position / length / self-preference) at *product-pipeline* altitude — taught here. Deep judge-vs-human calibration methodology is owned by `model-evaluation-engineer`.
- Trace-level instrumentation with OpenTelemetry Gen-AI / OpenInference conventions — taught here.
- Eval-gated CI/CD for prompts, chains, and agents — taught here.
- App-side safety and guardrail-effectiveness measurement (HarmBench shape, AgentDojo, InjecAgent, OWASP LLM Top 10 mapping) — taught here.
- Trajectory scoring, tool-call correctness, partial-credit rubrics — taught here.
- Online eval loop with drift / judge-drift alerts and sequential monitoring — taught here.
- NIST AI RMF / EU AI Act / ISO 42001 / ISO 25059 *interface* to release gates — taught here at the interface level; deeper governance shape is owned by the peer Governance-family track.
- **Model-eval methodology depth** (validity theory, benchmark construction, statistical estimation, cross-modality methodology, MLPerf-style serving benchmarks) — owned by [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30).
- **Release-assurance / governance shape** (audit trails, regulator-facing docs, third-party evaluator interface) — owned by [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (peer, level 25, Governance family) and [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning).
- **Alignment-risk methodology and red-team data generation** — owned by [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning).
- **Post-training depth** (SFT / PEFT / RLHF / DPO) — owned by [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning).
- **Distributed-training platform engineering** — owned by [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning).
- **Deep ML/AI security** (model extraction, eval-set exfiltration, judge supply-chain attacks, adversarial-eval depth) — owned by [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning).

## Hardware and credits

- Modules 101 and 109 can be done on a CPU laptop with a small classical-ML dataset.
- Module 102 (trace instrumentation) is API-bound plus a small local Docker for open-source observability backends (Phoenix / Langfuse can run in a single container). Plan $10-$20 in frontier-API credits.
- Module 103 (trajectory eval) is API- and Docker-bound (sandboxed agent replay). Plan $30-$60 in API credits and a machine that can spare 8-16 GB of RAM for the sandbox.
- Module 104 (LLM-as-judge) is API-bound. Plan $30-$60 across at least two judge tiers plus one open-source judge (Prometheus 2 or JudgeLM) hosted on Hugging Face Inference Endpoints or a rented 24-GB GPU.
- Module 105 (RAG eval) is API-bound plus a small local vector store; $20-$40 in API credits is enough.
- Module 106 (eval-gated CI/CD) is API-bound plus GitHub Actions minutes; the free GH tier is enough for the labs.
- Module 107 (online eval / regression) needs a small container running Phoenix / Langfuse / Weave and $30-$60 of API credits.
- Module 108 (app safety / guardrails) is API-bound with a small local guardrail model (Llama Guard 8B on a 16-GB GPU). **Do not collect or distribute attack content — use the referenced public benchmark suites in read-only form and delete after use.**
- Module 110 (eval-data-platform slice) needs a small cloud account for object storage + managed Postgres or DuckDB — under $10 for the lab.
- Module 111 (cost / latency / quality) is API-bound and needs a spread of tiers — plan $30-$60 across three model tiers.
- Module 112 (owning the program) is document-shaped and needs no compute.

<!-- needs-research: confirm the hardware-and-credits budget against the live posting evidence on the next autonomous research cycle; if postings consistently demand multi-GPU judge fine-tuning or larger self-hosted guardrail models, raise the floor on the recommended-hardware list. -->
