# AI Evaluation Engineer Curriculum

**Role level:** 30 (deep specialist — peer to `model-evaluation-engineer` on the ML Engineering ladder and to `senior-ml-engineer` on the same level)
**Family:** AI Engineering
**Status:** planned — modules and projects below are the planned scope authored from [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Lessons and projects will be drafted by subsequent autonomous content cycles.

## Overview

This track teaches AI-application evaluation end-to-end for a deep specialist: reading a product spec into concrete eval criteria → trace-level instrumentation across LLM apps and agents → trajectory and tool-call evaluation → LLM-as-judge inside product pipelines → RAG evaluation at the application layer → eval-gated CI/CD for prompts, chains, and agents → online evaluation and regression detection tied to product SLOs → app-side safety and guardrail-effectiveness measurement → human review workflows for product teams → an eval-data-platform slice inside an AI organisation → cost / latency / quality trade-off evaluation → owning the AI eval program across product, infra, safety, and governance.

It is a **specialist track at level 30**, positioned in the **AI Engineering family**. The differentiator against the peer `model-evaluation-engineer-learning` (ML Engineering family, level 30) is *application altitude and product shape*: this curriculum owns eval-engineering depth for AI **products** (prompts, chains, agents, RAG applications, judge pipelines, release gates, online eval loops). Model-eval methodology depth (validity theory, benchmark construction, judge-vs-human calibration methodology, MLPerf-style serving benchmarks) is owned by the peer track at the same level and *consumed* here.

Total planned commitment: **160 hours across 12 modules** + **130 hours across 3 projects** = **~290 hours**.

## Ownership rule

Following the project-wide ownership rule, this curriculum:

- **Owns** AI-application-shaped evaluation depth end-to-end — product-spec-to-eval-plan, trace instrumentation, trajectory / tool-call eval, LLM-as-judge inside product pipelines, RAG eval at the application layer, eval-gated CI/CD, online eval and regression detection, app-side safety and guardrail eval, human review workflows for product teams, an eval-data-platform slice, cost / latency / quality trade-off evaluation, and cross-team ownership of an AI eval program.
- **Defers down** to [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (level 20) for classical ML / PyTorch / sklearn-eval / packaging fundamentals; to [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning), [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning), and [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) for the systems being evaluated; to [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) for engineering-craft prerequisites.
- **Defers sideways** to [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30, ML Engineering family) for statistical methodology depth, benchmark construction depth, judge-vs-human calibration methodology depth, cross-modality model-eval methodology, and MLPerf-style serving benchmarks; to [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (peer, level 25, Governance family) for the release-assurance / audit-trail / regulator-facing shape; to [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning) (peer, level 30) for alignment-risk methodology, harm modelling, and red-team data generation; to [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning), [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning), [`nlp-engineer-learning`](https://github.com/ml-engineering-curriculum/nlp-engineer-learning), and [`ai-infra-mlops-learning`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) for their respective specialities.
- **Defers up** to [`senior-agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/senior-agentic-ai-engineer-learning), [`agentic-systems-architect-learning`](https://github.com/ai-engineering-curriculum/agentic-systems-architect-learning), [`staff-ml-engineer-learning`](https://github.com/ml-engineering-curriculum/staff-ml-engineer-learning), and [`principal-ml-engineer-learning`](https://github.com/ml-engineering-curriculum/principal-ml-engineer-learning) for architectural and leadership scope; to [`ai-infra-ml-platform-learning`](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning) for self-service ML/AI platform engineering; to [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) for deep ML/AI security (eval-set exfiltration, judge supply-chain, adversarial-eval depth).
- **Links out** to [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning) and [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning) for governance / compliance / model-card review depth.

See [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) for the requirements-to-coverage map and the cited public references the catalog is grounded in.

## Module Plan

| Module | Title | Hours | Status |
|---|---|---|---|
| mod-101-product-shaped-eval-foundations | Product-Shaped Evaluation Foundations: From Spec to Eval Plan | 10 | planned |
| mod-102-trace-instrumentation | Trace-Level Instrumentation for LLM Apps and Agents | 14 | planned |
| mod-103-trajectory-and-tool-eval | Trajectory and Tool-Call Evaluation for Production Agents | 16 | planned |
| mod-104-llm-as-judge-in-product | LLM-as-Judge in Product Pipelines: Rubrics, Bias Controls, and Judge-Tier Routing | 14 | planned |
| mod-105-rag-eval-app-layer | RAG Evaluation at the Application Layer | 12 | planned |
| mod-106-eval-gated-cicd | Eval-Gated CI/CD for Prompts, Chains, and Agents | 14 | planned |
| mod-107-online-eval-and-regression | Online Evaluation and Regression Detection at the App Layer | 16 | planned |
| mod-108-app-safety-and-guardrails-eval | App-Side Safety and Guardrail-Effectiveness Evaluation | 14 | planned |
| mod-109-human-review-workflows | Human Review Workflows for AI Product Teams | 10 | planned |
| mod-110-eval-data-platform-slice | Eval-Data-Platform Slice for AI Applications | 16 | planned |
| mod-111-cost-latency-quality-tradeoff | Cost / Latency / Quality Trade-off Evaluation | 12 | planned |
| mod-112-owning-an-ai-eval-program | Owning an AI Evaluation Program Across Product, Infra, Safety, and Governance | 12 | planned |

## Project Plan

| Project | Title | Hours | Status |
|---|---|---|---|
| project-101-product-agent-trajectory-eval-capstone | Product-Agent Trajectory Evaluation Capstone | 35 | planned |
| project-102-eval-gated-release-pipeline-rag-app | Eval-Gated Release Pipeline for a RAG Application | 40 | planned |
| project-103-ai-eval-platform-slice | AI Eval Platform Slice for a Product Family | 55 | planned |

## Module summaries

### mod-101 — Product-Shaped Evaluation Foundations
Translate a product spec into concrete eval criteria mapped to product SLOs; decide what belongs offline vs. online vs. in a governance packet; structure a per-surface eval plan using ISO/IEC 25059 quality dimensions and NIST AI RMF Measure as a scope frame; recognise the failure modes of leaderboard-shaped eval when it collides with a real product surface.

### mod-102 — Trace-Level Instrumentation
Instrument LLM applications and agents with OpenTelemetry GenAI / OpenInference conventions; wire Arize Phoenix, Langfuse, W&B Weave, Braintrust, and LangSmith end-to-end; design a span-attribute schema that captures prompt / response / tool calls / retrieved context / tokens / cost / latency / model version at debugging granularity; adopt a sampling and PII-scrubbing policy that keeps trace cost inside a product budget.

### mod-103 — Trajectory and Tool-Call Evaluation
Design trajectory-level scorers with per-step tool-call correctness, final-answer correctness, and partial credit; enforce cost / latency / step-count / retry-rate budgets; use Inspect (UK AISI) agent harness for product-shaped agent evals; adopt SWE-bench Verified, τ-bench, WebArena, GAIA, AgentBench, Berkeley Function-Calling Leaderboard, and ToolBench as *shape templates* for internal suites; design deterministic replay for failing trajectories with recorded tool responses and pinned model version.

### mod-104 — LLM-as-Judge in Product Pipelines
Design absolute and pairwise rubrics for real product criteria; apply position-bias (swap-and-average), length-bias, and self-preference controls inside the judge pipeline; use G-Eval-style chain-of-thought rubrics with DeepEval / Braintrust / Promptfoo; know when to switch to Prometheus 2 or JudgeLM for cost or reproducibility; quick-calibrate against a small human gold set and know when to escalate to `model-evaluation-engineer` for full calibration methodology (Cohen's kappa / Spearman / Kendall / bootstrap CIs); route across judge tiers inside a budget; monitor for judge drift when the judge model itself upgrades.

### mod-105 — RAG Evaluation at the Application Layer
Apply the RAG-triad (faithfulness / answer relevancy / context precision & recall) with RAGAS / TruLens / DeepEval; separate retrieval eval from generation eval so a regression can be attributed; design a product-shaped RAG suite for chat, search, or copilot surfaces with distractor documents and hallucination-tempting prompts; add noise-sensitivity and context-utilisation diagnostics; coordinate with the `rag-engineer` peer track — this module owns the eval, RAG engineering is owned there.

### mod-106 — Eval-Gated CI/CD for Prompts, Chains, and Agents
Wire evaluations into pull-request CI; snapshot and replay production traces as regression fixtures; register pre-registered pass/fail thresholds; build declarative eval configs with Promptfoo (YAML), DeepEval pytest hooks, Braintrust / W&B Weave / Langfuse CI runners; design deployment gates (offline pass → canary → progressive rollout) and the interface to the deploy pipeline the platform team owns.

### mod-107 — Online Evaluation and Regression Detection
Design a sampled-trace online-eval loop that scores live traffic on quality / safety / cost / latency within budget; detect input / output / judge drift; wire canary and shadow launches with per-cohort quality and safety gates; apply sequential testing / confidence sequences for continuous monitoring; interface with the A/B platform (CUPED and pre-registration contract) knowing that deep experimental methodology is owned by `model-evaluation-engineer` / `senior-ml-engineer`; build dashboards a PM, an on-call engineer, and a safety reviewer can each read.

### mod-108 — App-Side Safety and Guardrails Eval
Measure jailbreak-resistance on the *actual application surface* with HarmBench-style suites (without leaking payloads into the repo); measure prompt-injection robustness in RAG and tool-calling agents with AgentDojo / InjecAgent; measure tool-abuse / unintended-tool-use with sandboxed replay; measure guardrail effectiveness (NeMo Guardrails, Guardrails AI, Llama Guard, provider moderation) with false-positive / false-negative reporting and cost accounting; measure refusal / over-refusal at the app surface; map findings to OWASP Top 10 for LLM Applications, MITRE ATLAS, Google SAIF; know the delegation boundary to `ai-risk-engineer` (harm models / red-team data) and to `model-evaluation-engineer` (frontier-model dangerous-capability evals).

### mod-109 — Human Review Workflows for AI Product Teams
Stand up expert-review workflows in Argilla or Label Studio integrated with the trace store; capture and triage thumbs-up/down and structured user feedback; compute practical inter-annotator agreement at product scale; rotate gold sets in step with product-surface changes; choose vendor vs. in-house review for a given surface — deferring vendor-methodology depth to `model-evaluation-engineer`.

### mod-110 — Eval-Data-Platform Slice
Design a trace warehouse + eval-result store with lineage across model / prompt / chain / retriever / decoding / seed / eval-set hash; build a test-case management layer product teams can contribute to without touching harness code; orchestrate multiple runners (Promptfoo, DeepEval, RAGAS, Braintrust, Weave, custom) behind one plane; wire the eval platform into CI and into the online-eval loop with per-team cost accounting; instrument the eval platform itself with SLO / SLA so it's operable, not a script.

### mod-111 — Cost / Latency / Quality Trade-off Evaluation
Design multi-objective eval reports (quality vs. cost vs. latency vs. safety) that let product decide what to ship; evaluate model routing (frontier vs. mid-tier vs. open-source, classifier-gated); run distillation / replacement regressions; measure app-altitude latency the way users experience it (TTFT, TPOT, streaming p50/p95); track token accounting per feature so an eval-driven cost report drives product decisions.

### mod-112 — Owning an AI Evaluation Program
Translate a product roadmap into a release-gate architecture; author delegation contracts to peer specialists (`model-evaluation-engineer`, `ai-risk-engineer`, `ai-evaluation-engineer` / `ai-governance-analyst`, `ai-infra-security`, `ai-infra-mlops` / `ai-infra-ml-platform`); make the build-vs-buy decision across app-eval platforms (Weave, Phoenix, Langfuse, Braintrust, Promptfoo, DeepEval, RAGAS, Humanloop, Galileo, Patronus, Vertex Eval, Bedrock Eval, Azure AI Foundry Eval); run incident-driven eval investment; compose a product-side model-card slice from the eval evidence; map to NIST AI RMF Measure/Manage, EU AI Act general-purpose AI obligations, and ISO/IEC 42001 controls — knowing governance shape is owned by the peer Governance-family track.

## Assessment

Each module ships **1 quiz** plus the exercises and a lab listed in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Each project ships a portfolio-grade README, a reproducibility bundle (config + dataset / model / prompt / retriever / decoding / seed hashes + judge prompts), and an explicit rubric covering product-fit, trace-instrumentation quality, judge design, safety coverage, CI/release integration, and documentation.

## Where to go after this curriculum

- **`senior-agentic-ai-engineer-learning`** — next step on the AI Engineering ladder for architectural / multi-team scope.
- **`agentic-systems-architect-learning`** (level 48) — architectural depth for agentic systems that consumes this curriculum's eval methodology.
- **`staff-ml-engineer-learning` / `principal-ml-engineer-learning`** — for cross-team / architectural scope on the ML ladder.
- **`model-evaluation-engineer-learning`** — peer specialist at the same level that owns model-eval methodology depth; a natural cross-training track.
- **`ai-evaluation-engineer-learning`** — peer specialist (Governance family, level 25) for release-assurance / regulator-facing shape.
- **`ai-risk-engineer-learning`** — peer specialist (level 30) for alignment-risk methodology and red-team data generation.
- **`ai-infra-ml-platform-learning`** — for self-service ML/AI platform engineering hosting eval-as-a-service at multi-tenant scale.
- **`ai-infra-security-learning`** — for deep ML/AI security, including eval-set integrity and judge supply-chain.
- **`ai-governance-analyst-learning`** / **`head-of-ai-governance-learning`** — for governance / compliance / model-card review depth.

<!-- needs-research: backfill industry-frequency evidence into JOB_REQUIREMENTS.md once the autonomous research loop runs with WebSearch / WebFetch exercised against live job boards; demote any module or exercise whose underlying requirement does not show up in ≥3 in-window postings. -->
