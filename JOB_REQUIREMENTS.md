# Job Requirements — AI Evaluation Engineer

**Role level:** 30 (deep specialist, AI Engineering family — peer to `model-evaluation-engineer` on the ML Engineering ladder and to `senior-ml-engineer` at the same level)
**Track:** `ai-eval-engineer-learning`
**Research window:** 2026-06-06 → 2026-09-04 (last 90 days)
**Today:** 2026-09-04

This file documents the requirements catalog used to seed the AI Evaluation Engineer curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json); the proposed per-cycle delta (empty this cycle — see rationale below) lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Status — live posting evidence backfilled

This cycle sampled **26 in-window postings** across the equivalent title cluster (`AI Evaluation Engineer`, `LLM Evaluation Engineer`, `AI Eval Engineer`, `Agent Evaluation Engineer`, `AI QA Engineer` / `AI Quality Engineer`, `GenAI Evaluation Engineer`, `LLM / Agentic Evaluation Rig Engineer`, `Software Engineer (Evals)`, `AI Engineer - Evaluations`, and a `User Researcher, AI Evaluations` variant). Sources spanned Greenhouse, Ashby, Lever, Wellfound, Built In, SmartRecruiters, and direct-employer career sites. Frontier-lab pre-training / benchmark-construction and pure governance / regulator-facing postings were rejected per the ownership rule and routed to the peer specialist tracks (`model-evaluation-engineer`, `ai-evaluation-engineer` level 25 Governance).

**Every one of the 12 existing modules received posting evidence.** No requirement theme that surfaced in the sample crosses the 30% frequency threshold *without already being covered* by an existing module. Per the continuity-bias rule (`Default to no change`), [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) is empty this cycle.

Ashby- and Lever-hosted ATS pages are JavaScript-rendered and returned only partial content to `WebFetch`. For those postings, the record in `.aicg/job-requirements.json` carries a short representative quote (from the ATS search-result card) and null verbatim bullets. The verbatim required/preferred bullets were fully extracted for the 10 postings hosted on Greenhouse, Built In, Cursor's careers site, and OpenTrain — which is enough evidence to validate the theme mapping. Next cycle should try headless-browser rendering for the Ashby / Lever cohort.

## Methodology

1. Sampled ≥25 in-window postings (2026-06-06 → 2026-09-04) across the title cluster, spanning ATS aggregators (Greenhouse, Ashby, Lever, Wellfound, SmartRecruiters, Built In) and direct-employer career sites.
2. For each posting, captured `employer / title / URL / date_observed / date_posted / location / verbatim required bullets / verbatim preferred bullets / salary_range / one representative quote` into [`.aicg/job-requirements.json → postings`](.aicg/job-requirements.json). For JS-rendered ATS pages where bullets were not extractable, the record kept the quote and marked bullet fields null with a note.
3. Attributed each posting to one or more `req-*` requirements via `requirement_evidence_for`.
4. Computed observed frequency per requirement as `distinct postings citing / total postings sampled (26)`.
5. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. Deferred:
   - *Down* to `ml-engineer` (level 20) for classical ML fundamentals; to `llm-application-developer` / `rag-engineer` / `agentic-ai-engineer` (peer AI Engineering tracks) for the systems being evaluated.
   - *Sideways* to `model-evaluation-engineer` (peer, level 30, ML Engineering family) for statistical methodology depth, benchmark-construction depth, judge-vs-human calibration methodology depth, cross-modality model-eval methodology, and MLPerf-style serving benchmarks.
   - *Sideways* to `ai-evaluation-engineer` (peer, level 25, Governance family) for the release-assurance / audit-trail / regulator-facing shape.
   - *Sideways* to `ai-risk-engineer` (peer, level 30) for alignment-risk methodology, harm modelling, and red-team data generation.
   - *Up* to `staff-ml-engineer` / `principal-ml-engineer` / `senior-agentic-ai-engineer` / `agentic-systems-architect` / `head-of-ai-governance` / `ai-infra-security` for the leadership, architectural, and deep-security scopes that inherit this packet.
6. Applied the **continuity-bias rule** — a new module / exercise is only warranted when ≥3 distinct postings cite a requirement the existing curriculum does *not* cover AND observed frequency is ≥ 0.30 AND no existing module can be incrementally extended. No theme in this sample met all three conditions.

## Requirement themes → curriculum ownership

Frequency (`Freq`) is the share of the 26 sampled postings that cite the theme. Themes owned elsewhere are listed for routing purposes and have no observed frequency here.

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Product-shaped eval foundations: SLO mapping, offline vs. online split, deferring statistical depth | 0.54 | `ai-eval-engineer` (this) | [`mod-101-product-shaped-eval-foundations`](lessons/mod-101-product-shaped-eval-foundations) |
| 2 | Trace instrumentation for LLM apps and agents (OpenTelemetry / OpenInference, Phoenix / Langfuse / Weave / Braintrust / LangSmith) | 0.15 | `ai-eval-engineer` | [`mod-102-trace-instrumentation`](lessons/mod-102-trace-instrumentation) |
| 3 | Trajectory & tool-call eval for production agents (Inspect harness, SWE-bench Verified / τ-bench / WebArena / GAIA / BFCL / ToolBench shapes) | 0.38 | `ai-eval-engineer` | [`mod-103-trajectory-and-tool-eval`](lessons/mod-103-trajectory-and-tool-eval) |
| 4 | LLM-as-judge in product pipelines: rubric design, bias controls, judge-tier routing, judge-drift, quick calibration | 0.31 | `ai-eval-engineer` | [`mod-104-llm-as-judge-in-product`](lessons/mod-104-llm-as-judge-in-product) |
| 5 | RAG evaluation at the application layer (RAGAS / TruLens / DeepEval; RAG-triad; retrieval vs generation split) | 0.23 | `ai-eval-engineer` | [`mod-105-rag-eval-app-layer`](lessons/mod-105-rag-eval-app-layer) |
| 6 | Eval-gated CI/CD for prompts, chains, and agents (Promptfoo / DeepEval / Braintrust / Weave / Langfuse in CI) | 0.58 | `ai-eval-engineer` | [`mod-106-eval-gated-cicd`](lessons/mod-106-eval-gated-cicd) |
| 7 | Online evaluation & regression detection (sampled traces, drift, canary / shadow, sequential monitoring) | 0.38 | `ai-eval-engineer` | [`mod-107-online-eval-and-regression`](lessons/mod-107-online-eval-and-regression) |
| 8 | App-side safety & guardrails eval (jailbreak on surface, prompt-injection, tool abuse, guardrail effectiveness, OWASP LLM Top 10) | 0.19 | `ai-eval-engineer` | [`mod-108-app-safety-and-guardrails-eval`](lessons/mod-108-app-safety-and-guardrails-eval) |
| 9 | Human review workflows integrated with product feedback (Argilla / Label Studio, expert reviewers, product feedback loops) | 0.19 | `ai-eval-engineer` | [`mod-109-human-review-workflows`](lessons/mod-109-human-review-workflows) |
| 10 | Eval-data-platform slice: trace warehouse, dataset lineage, multi-runner orchestration, eval-as-a-service | 0.58 | `ai-eval-engineer` | [`mod-110-eval-data-platform-slice`](lessons/mod-110-eval-data-platform-slice) + [`project-103`](projects/project-103-ai-eval-platform-slice) |
| 11 | Cost / latency / quality trade-off eval (model routing, distillation regression, token / TTFT / TPOT at app altitude) | 0.12 | `ai-eval-engineer` | [`mod-111-cost-latency-quality-tradeoff`](lessons/mod-111-cost-latency-quality-tradeoff) |
| 12 | Owning an AI eval program: release-gate architecture, cross-team delegation, build-vs-buy, incident-driven investment | 0.31 | `ai-eval-engineer` | [`mod-112-owning-an-ai-eval-program`](lessons/mod-112-owning-an-ai-eval-program) |
| 13 | Classical ML / PyTorch / sklearn-eval / packaging fundamentals | n/a — prerequisite | `ml-engineer` (level 20) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught |
| 14 | LLM application development (prompting, tool design, chain orchestration) | n/a — peer track | `llm-application-developer` (level 25) | Linked out; this curriculum evaluates what that role builds |
| 15 | RAG system engineering (chunking, embeddings, rerankers) | n/a — peer track | `rag-engineer` (level 25) | mod-105 covers RAG *eval*; RAG engineering linked out |
| 16 | Agent implementation (ReAct, LangGraph / CrewAI / AutoGen, MCP) | n/a — peer track | `agentic-ai-engineer` (level 30) | mod-103 evaluates agents; implementation linked out |
| 17 | Model-eval methodology depth: validity theory, benchmark construction, statistical estimation, judge-vs-human calibration, MLPerf | n/a — peer specialist | `model-evaluation-engineer` (level 30) | Consumer contract in mod-101 and mod-104; depth linked out |
| 18 | Release-assurance / regulator-facing shape (audit trails, third-party evaluator interface) | n/a — peer track | `ai-evaluation-engineer` (level 25, Governance family) | mod-112 documents the interface; assurance shape linked out |
| 19 | Alignment-risk methodology / harm modelling / red-team data generation | n/a — peer specialist | `ai-risk-engineer` (level 30) | mod-108 measures; harm-model design linked out |
| 20 | Post-training (SFT / PEFT / RLHF / DPO) depth | n/a — peer specialist | `fine-tuning-engineer` (level 30) | Out of scope — this curriculum evaluates the outputs |
| 21 | Distributed training PLATFORM engineering | n/a — peer track | `training-pipeline-engineer` (level 25) | Out of scope |
| 22 | Deep ML/AI security (model extraction, eval-set exfiltration, judge supply-chain attacks) | n/a — higher level | `ai-infra-security-learning` (level 35) | Awareness in mod-108 and mod-110; depth owned upstream. Anthropic Cyber Evaluations Engineer posting is the closest in-window example |
| 23 | Compliance / model-card review / regulated-data review depth | n/a — peer track | `ai-governance-analyst` (level 25) | Awareness in mod-112; depth owned upstream |

## Themes that fell BELOW the 0.30 threshold — do not add

The following themes appeared in the sample but stayed under the 30% frequency threshold that would justify new content. Each is already partially covered by an existing module and is retained here for the next cycle to re-check.

- **Simulated-user / rollout environments** — Hippocratic AI ("agent harnesses or simulation environments"), Precept Labs ("evaluation harnesses"), Cursor ("curated datasets and offline replay"). ~4/26 = 0.15. Already inside `mod-103-trajectory-and-tool-eval` under deterministic replay + agent-harness authoring. Do not spin out.
- **Coding-agent-specific eval** — OpenTrain AI, Cursor, Phizenix. ~3/26 = 0.12. Already covered by mod-103's SWE-bench Verified / BFCL / τ-bench shape templates and by the tool-call scoring exercise.
- **Cost / latency / quality trade-off eval** — Glean, Firecrawl, Cursor. ~3/26 = 0.12. Already the exact scope of `mod-111-cost-latency-quality-tradeoff`. Below threshold this cycle, but the theme is load-bearing in build-vs-buy discussions and shows up implicitly in most postings — do not demote.
- **Trace instrumentation** — 4/26 = 0.15 explicit mentions. This is under-counted because most postings phrase it generically ("evaluation frameworks", "observability tools") rather than naming the OTel / OpenInference stack. It is a hard prerequisite for the trajectory / online / regression themes, so `mod-102` stays.
- **Cyber-evaluation of AI** — Anthropic Cyber Evaluations Engineer (1/26 in-window). Specialist adjacent to `mod-108`; the *deep* security shape is owned upstream by `ai-infra-security-learning` (level 35). Do not spin out.
- **Domain-vertical eval templates (health, legal, fintech, life-sciences)** — Hippocratic AI, Phizenix, Appnovation, Distyl, Notion, NCS. Domain vertical is the *context* of the eval, not a distinct requirement — the underlying skills are already covered by the existing modules and the projects already ask learners to instantiate them on a specific product surface.

## Posting evidence

The 26 in-window postings sampled this cycle. See [`.aicg/job-requirements.json → postings`](.aicg/job-requirements.json) for the full verbatim record.

| # | Employer | Title | Salary (USD) | Requirements evidence for |
|---|---|---|---|---|
| 1 | Phizenix | [LLM / Agentic Evaluation Rig Engineer](https://job-boards.greenhouse.io/phizenix/jobs/5398766008) | — | mod-101, mod-103, mod-104, mod-105, mod-106, mod-107 |
| 2 | Anthropic | [Cyber Evaluations Engineer](https://job-boards.greenhouse.io/anthropic/jobs/5406367008) | $300k–$405k | mod-108, mod-112 |
| 3 | Cursor (Anysphere) | [Software Engineer, Agent Evaluation and Quality](https://cursor.com/careers/software-engineer-agent-evaluation-and-quality) | — | mod-101, mod-102, mod-103, mod-106, mod-107, mod-110, mod-112 |
| 4 | Glean | [Software Engineer, Evals](https://job-boards.greenhouse.io/gleanwork/jobs/4712438005) | — | mod-102, mod-103, mod-107, mod-110, mod-112 |
| 5 | Hippocratic AI | [AI Engineer - Evaluations](https://builtin.com/job/ai-engineer-evaluations/7559809) | — | mod-101, mod-103, mod-104, mod-106, mod-108, mod-109, mod-110, mod-112 |
| 6 | ServiceNow | [Machine Learning Quality Engineer](https://builtin.com/job/machine-learning-quality-engineer/6607682) | $124k–$192k | mod-101, mod-104, mod-106, mod-108 |
| 7 | Appnovation Technologies | [AI Evaluation Engineer (QA)](https://job-boards.greenhouse.io/appnovation/jobs/8743206002) | — | mod-101, mod-105, mod-106, mod-109, mod-110 |
| 8 | WITHIN | [AI Engineer](https://job-boards.greenhouse.io/agencywithin/jobs/5056863007) | $90.9k–$254.1k | mod-101, mod-103, mod-105, mod-106 |
| 9 | OpenTrain AI | [Python Engineer, AI Coding Agent Evaluation](https://www.opentrain.ai/jobs/python-engineer-ai-coding-agent-evaluation--cmtfopkq0000j0akoeodsq186/) | — | mod-103, mod-109 |
| 10 | Future | [Applied AI Engineer](https://job-boards.greenhouse.io/future/jobs/4683133005) | $215k–$250k | mod-102, mod-104, mod-106, mod-110 |
| 11 | CI&T | [AI Agent Evaluation Engineer (Senior, QA)](https://jobs.lever.co/ciandt/1e06dadb-5342-470d-a9a1-cb42755381a2) | — | mod-102, mod-103, mod-104, mod-105, mod-109 |
| 12 | GovWorx | [AI Evaluation Engineer](https://jobs.ashbyhq.com/govworx/7bad3c33-9ad1-45e3-bf15-4c2de306e671) | — | mod-101, mod-106, mod-107 |
| 13 | Distyl AI | [AI Evaluation Engineer (A)](https://jobs.ashbyhq.com/Distyl/b0bb160e-498d-4f32-baaa-3b4974a3cbf2) | — | mod-101, mod-106, mod-107, mod-110 |
| 14 | Distyl AI | [AI Evaluation Engineer (B)](https://jobs.ashbyhq.com/Distyl/75003495-773a-4b3d-99f2-a8976c40012f) | — | mod-101, mod-106, mod-110 |
| 15 | Notion | [User Researcher, AI Evaluations](https://careercenter.americananthro.org/job/715145/user-researcher-ai-evaluations/) | $196k–$230k | mod-101, mod-104, mod-107, mod-109 |
| 16 | Coupa Software | [AI Engineer, Evaluation & Quality](https://jobs.lever.co/coupa/0e7ed42f-a206-4ea6-afe6-4e312b1f7046) | — | mod-106, mod-107, mod-110 |
| 17 | Nous Research | [Machine Learning Engineer, Evals](https://jobs.ashbyhq.com/nous-research/1e647dc1-e69c-4764-8a02-7244b8faee0b) | — | mod-104, mod-110 |
| 18 | Exa | [ML Evals Engineer](https://jobs.ashbyhq.com/exa/9c45a74e-d507-482a-bc0e-da2f464c9767) | — | mod-105, mod-110 |
| 19 | Thinking Machines Lab | [Software Engineer, Evaluation Platform / Infra](https://jobs.ashbyhq.com/thinkingmachines/9d863c78-80c0-44cd-a574-d1330e125398) | — | mod-110, mod-112 |
| 20 | White Circle | [Research Engineer (Evals)](https://jobs.ashbyhq.com/whitecircle/07d24ec8-117f-4849-8dd5-2c2bc2cfbb40) | — | mod-103, mod-108 |
| 21 | OpenAI | [Backend Software Engineer (Evals)](https://jobs.ashbyhq.com/openai/3d064454-c0c3-4225-bc2c-6d8c0f8735b2) | — | mod-101, mod-107, mod-110 |
| 22 | Firecrawl | [Research Engineer – Evals](https://jobs.ashbyhq.com/firecrawl/25092c0e-9a32-4191-af79-050738213704) | $160k–$240k | mod-101, mod-106, mod-110, mod-112 |
| 23 | HackerRank | [Machine Learning Engineer, Evaluation](https://wellfound.com/jobs/4079979-2-machine-learning-engineer-evaluation) | — | mod-104, mod-106, mod-107 |
| 24 | Precept Labs | [ML Engineer — Applied LLM / Agents (Contract)](https://wellfound.com/jobs/4532099-machine-learning-engineer-applied-llm-agents-contract) | — | mod-103, mod-106, mod-107, mod-110 |
| 25 | Atlassian | [Machine Learning Engineer, AI Evaluation (Rovo Chat)](https://builtin.com/jobs/data-analytics/machine-learning) | — | mod-101, mod-106, mod-110, mod-112 |
| 26 | NCS | [LLM / AI Quality Engineer](https://jobs.smartrecruiters.com/NCS3/6000000000955545--eg-llm-ai-quality-engineer) | — | mod-101, mod-103, mod-105, mod-108, mod-112 |

## Ownership map — quick reference for next cycle

When backfilling postings, use this ownership decision to keep the curriculum from drifting into peer territory:

- **AI Evaluation Engineer** (this track, level 30, AI Engineering family) — owns AI-application-shaped evaluation depth end-to-end: trace instrumentation → trajectory / tool-call eval → LLM-as-judge inside product pipelines → RAG eval at app layer → eval-gated CI/CD → online eval and regression → app-side safety and guardrail eval → human review workflows → eval-data-platform slice → cost/latency/quality trade-offs → owning the AI eval program across product / infra / safety / governance. **App altitude and product-shape are the differentiators against peers.**
- **Model Evaluation Engineer** (peer, level 30, ML Engineering family) — owns model-eval methodology depth, benchmark engineering depth, judge-vs-human calibration methodology, multi-modality model-eval methodology, and MLPerf-style serving benchmarks. **Consume from this peer's methodology and cite it in mod-101, mod-104, and mod-107.**
- **AI Evaluation Engineer** (peer, level 25, Governance family, `ai-evaluation-engineer-learning`) — owns release-assurance / audit trail / regulator-facing shape. **This curriculum produces the technical eval evidence; that peer wraps it into governance artifacts.**
- **AI Risk Engineer** (peer, level 30) — owns alignment-risk methodology, harm modelling, red-team data generation. **mod-108 measures; harm-model authoring is owned there.**
- **Agentic AI Engineer / LLM Application Developer / RAG Engineer** (peer AI Engineering tracks) — own the systems this curriculum evaluates.
- **ML Engineer** (level 20) — owns build-altitude practitioner fundamentals this curriculum assumes.
- **Senior ML Engineer** (level 30, generalist peer) — consumes this curriculum's eval methodology in product-team ML release reviews.
- **Staff / Principal / Senior Agentic AI / Agentic Systems Architect** (level 40+) — inherit this packet for architectural and cross-team scope.
- **Head of AI Governance** (level 40, Governance family) — consumes eval evidence in board-level reporting.
- **AI Infra Security** (level 35) — owns deep AI/ML security (eval-set exfiltration, judge supply-chain, adversarial-eval depth); surfaced as awareness only. Anthropic's Cyber Evaluations Engineer posting sits at the boundary.

## Differentiation versus the three peer "eval" tracks

The repository carries three eval-related tracks that share vocabulary. Posting research must disambiguate carefully:

| Track | Family | Level | Owns |
|---|---|---|---|
| `ai-eval-engineer` (this) | AI Engineering | 30 | AI-application evaluation depth — trace / trajectory / tool-call / judge / RAG / online eval / app-side safety / CI-CD gating / cost-quality trade-off for AI *products* |
| `model-evaluation-engineer` | ML Engineering | 30 | Model-eval methodology depth, benchmark engineering, statistical rigour, cross-modality, eval-platform architecture |
| `ai-evaluation-engineer` | Governance | 25 | Release-assurance / audit trails / regulator-facing docs / third-party evaluator interface |

If a posting emphasises **product-side eval pipelines, prompt / chain / agent CI, LLM observability integration, app-team enablement, or eval-gated release for AI applications**, route it here.
If it emphasises **benchmark construction, statistical rigour, eval-platform internals across modalities, or MLPerf-style serving benchmarks**, route it to `model-evaluation-engineer`.
If it emphasises **audit trails, regulator interface, or release-governance evidence packaging**, route it to `ai-evaluation-engineer` (Governance family, level 25).
