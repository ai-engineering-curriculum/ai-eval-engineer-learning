# AI Engineering · AI Evaluation Engineer — Learning Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum). Live cohorts &amp; team programs: **[ai-infra-curriculum.github.io](https://ai-infra-curriculum.github.io/)**.
<!-- /aicg:site-banner -->

<!-- aicg:sponsor -->
> 💜 **[Sponsor this curriculum](https://github.com/sponsors/ai-engineering-curriculum)** — sponsorships keep the whole open-source AI Career Curriculum free and moving.
<!-- /aicg:sponsor -->

Evaluate AI applications end-to-end: trace-level observability, trajectory and tool-call evaluation, LLM-as-judge in product pipelines, RAG evaluation, eval-gated CI/CD, online eval and regression detection, app-side safety and guardrail measurement, and eval-platform ownership across product, infra, safety, and governance.

> **Status**: curriculum plan authored (2026-07-08). Lessons and project READMEs will be drafted by subsequent autonomous content cycles. See [`CURRICULUM.md`](CURRICULUM.md) for the planned scope and [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) for the requirements-to-coverage map (postings list deferred to the next research cycle — see `JOB_REQUIREMENTS.md` Status section).

**Level:** 30 (deep specialist — peer to `model-evaluation-engineer` on the ML Engineering ladder and to `senior-ml-engineer` at the same level).
**Family:** AI Engineering.
**Planned scope:** 12 modules (160h) + 3 projects (130h) = ~290 hours.

## What this track owns

AI-application-shaped evaluation depth end-to-end:

- Product-shaped eval foundations — spec → criteria → offline / online split, SLO mapping, ISO/IEC 25059 & NIST AI RMF framing
- Trace-level instrumentation across LLM apps and agents (OpenTelemetry GenAI / OpenInference; Arize Phoenix, Langfuse, W&B Weave, Braintrust, LangSmith)
- Trajectory and tool-call evaluation for production agents (Inspect harness; SWE-bench Verified / τ-bench / WebArena / GAIA / BFCL / ToolBench shape templates)
- LLM-as-judge in product pipelines — rubric design, bias controls, judge-tier routing, judge-drift monitoring
- RAG evaluation at the application layer — RAGAS / TruLens / DeepEval RAG-triad; retrieval vs. generation attribution
- Eval-gated CI/CD for prompts, chains, and agents — Promptfoo / DeepEval / Braintrust / Weave / Langfuse in CI with pre-registered thresholds
- Online evaluation and regression detection — sampled-trace eval, drift alerts, canary / shadow, sequential monitoring
- App-side safety and guardrails eval — HarmBench-shaped surface tests, AgentDojo / InjecAgent prompt injection, tool-abuse, guardrail effectiveness (NeMo Guardrails, Guardrails AI, Llama Guard, provider moderation), OWASP LLM Top 10 mapping
- Human review workflows integrated with product feedback (Argilla / Label Studio, expert review, user feedback triage)
- An eval-data-platform slice inside an AI product organisation (trace warehouse, lineage, multi-runner orchestration, per-team cost accounting, SLO on the eval system itself)
- Cost / latency / quality trade-off evaluation — model routing, distillation regression, TTFT / TPOT at app altitude
- Owning an AI evaluation program across product, infra, safety, and governance — release-gate architecture, delegation contracts, build-vs-buy, incident-driven investment, product-side model-card slice

## What this track defers

- Build-altitude ML / PyTorch / classical-eval fundamentals → [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (level 20).
- LLM application development / RAG engineering / agent implementation → [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning), [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning), [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) (peer AI Engineering tracks).
- Model-eval methodology depth (validity theory, benchmark construction, statistical estimation, cross-modality methodology, judge-vs-human calibration methodology, MLPerf-style serving benchmarks) → [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30, ML Engineering family).
- Release-assurance / governance shape (audit trails, regulator-facing docs, third-party evaluator interface) → [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (peer, level 25, Governance family).
- Alignment-risk methodology, harm modelling, and red-team data generation → [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning) (peer, level 30).
- Post-training depth (SFT / PEFT / RLHF / DPO) → [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning).
- Distributed-training PLATFORM engineering → [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning).
- Deep ML/AI security (model extraction, eval-set exfiltration, judge supply-chain attacks, adversarial-eval depth) → [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning).
- Governance / policy / model-card review depth → [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning), [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning).

## Layout

```
ai-eval-engineer-learning/
├── .aicg/                    curriculum-plan.json and job-requirements.json (machine-readable catalog)
├── lessons/mod-XXX-*/        modules with lectures, exercises, labs, quizzes (to be drafted)
├── projects/project-XXX-*/   multi-module capstones (to be drafted)
├── CURRICULUM.md             role-level coverage map
├── JOB_REQUIREMENTS.md       requirements catalog with citations and ownership map
├── PREREQUISITES.md          assumed entry skills
├── VERSIONS.md               release history
└── README.md                 this file
```

## Paired Solutions Repo

[`ai-eval-engineer-solutions`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-solutions) carries the reference implementations.

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
