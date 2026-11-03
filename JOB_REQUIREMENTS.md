# Job Requirements — AI Evaluation Engineer

**Role level:** 30 (deep specialist, AI Engineering family — peer to `model-evaluation-engineer` on the ML Engineering ladder and to `senior-ml-engineer` at the same level)
**Track:** `ai-eval-engineer-learning`
**Research window:** 2026-08-05 → 2026-11-03 (last 90 days)
**Today:** 2026-11-03

This file documents the requirements catalog used to seed the AI Evaluation Engineer curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json); the proposed per-cycle delta (empty this cycle — see rationale below) lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Status — 2026-11 refresh, continuity-bias upheld

This cycle resampled **29 in-window postings** across the equivalent title cluster (`AI Evaluation Engineer`, `AI Eval Engineer`, `LLM Evaluation Engineer`, `Agent Evaluation Engineer`, `AI QA Engineer` / `AI Quality Engineer`, `GenAI Evaluation Engineer`, `Software Engineer (Evals)`, `AI Engineer, Evaluations`, `Senior AI Quality Engineer`, `Staff SDET (GenAI/LLM Evaluation)`, `ML/MLE AI Evaluation`, `Research Engineer, Evals`). **10 of the 29 are newly surfaced this cycle** (Workday Senior AI Evaluation Engineer, ServiceNow Moveworks Staff SE Agent Eval Platform, Cortea AI Senior/Staff AI Engineer Quality & Evals, Apple Senior SDET LLM Evaluation & Automation, Judi Health AI Evaluation Engineer, Docket AI Evals Engineer, Mistral Evaluation Engineer Applied AI, Rubrik Senior AI Evals Engineer, Anthropic Applied AI Engineer Beneficial Deployments, Datadog Engineering Manager I AI Platform – Evaluation & Annotation). **19 of the Oct 2026 cycle's 28 counted postings were re-verified as still live or indeterminate** and are retained in current-window math; **9 are confirmed expired / 404** (Cursor Agent Eval Quality, Hippocratic AI Built In, GovWorx plain URL, Distyl B, OpenAI Backend Evals, Precept Labs, Hippocratic Senior MLE, Anomali, Block) and are excluded from current-window frequency math while retained in the historical record. Posting-level attrition in the Ashby/Lever/Wellfound cohort is substantial (~40–50% over 30 days), consistent with ATS auto-expiry; the retained cohort skews toward Greenhouse, Built In, SmartRecruiters, direct-employer career pages, and the Workday-native board. Sources this cycle spanned Greenhouse, Ashby, Lever, Workday-native, Dreamwork, Arbeitnow, AnitaB, General Catalyst portfolio-careers, Foundation Capital portfolio-careers, OpenTalent, SmartRecruiters, and Built In. Frontier-lab pre-training / benchmark-construction (METR, retained Oct listing) and pure governance / regulator-facing postings continue to be routed to peer specialist tracks (`model-evaluation-engineer`, `ai-evaluation-engineer` level 25 Governance) per the ownership rule.

**All 12 existing modules received evidence support this cycle.** **No requirement theme that surfaced crosses the 30% frequency threshold *without already being covered* by an existing module.** Current-window frequency structure (n=29): LLM-as-judge in product **0.59**, trajectory & tool-call eval **0.52**, eval-gated CI/CD **0.48**, eval-data-platform slice **0.45**, owning an AI eval program **0.41**, product-shaped eval foundations **0.38**, online eval & regression **0.34**, app-side safety & guardrails **0.34**, trace instrumentation **0.31**, RAG eval **0.31**, human review workflows **0.31**, cost / latency / quality trade-off **0.03**. **Ten of twelve themes are at or above the 0.30 gate** this cycle (up from seven in Oct), driven by the judge-and-trajectory-weighted new postings and by the removal of expired entries that previously anchored lower-frequency themes. Per the continuity-bias rule (`Default to no change`), [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) is empty this cycle.

**Watch list for next cycle:**

- **MCP server / tool-registry eval** — Workday Senior AI Evaluation Engineer explicitly lists `Model Context Protocol (MCP) and agent execution harnesses` as a preferred requirement. This is the first mainstream app-altitude AI-eval posting to name MCP in bullets rather than only in agent-engineer JDs. 1/29 = 0.03 — far below gate but a notable precursor pattern.
- **Long-horizon / multi-turn trajectory eval** — Workday (`synthetic multi-turn trajectory`), ServiceNow Moveworks (`real, irreversible actions in stateful, multi-tenant enterprise environments`), Rubrik (`end-to-end task completion rate`). 3/29 = 0.10 explicit. Already inside `mod-103` scope under deterministic replay + agent-harness authoring; if it crosses the 0.30 gate next cycle, extend `mod-103` with a long-horizon exercise before considering a new module.
- **Evals-driven synthetic data generation at app altitude** — Workday explicit (`synthetic multi-turn trajectory ... eval suites for release gates`), implicit in Netomi, Judi Health, Cortea. ~3–4/29 = 0.10–0.14. Already partially covered by `mod-109` (dataset authoring + labelling) and `mod-110` (dataset lineage). Still below gate.
- **Multimodal / voice-agent eval at app altitude** — Judi Health (voice/call-center transcripts, nice-to-have), Apple SDET (image/visual generation quality judging). 2/29 = 0.07. Owned at model-eval altitude by `model-evaluation-engineer`; watch whether product-side eval demand grows.
- **Agent-memory eval**, **evaluator-model fine-tuning**, **reasoning / chain-of-thought faithfulness eval** — 0/29 explicit mentions in app-altitude postings this cycle.

Ashby-, Lever-, Workday-native-, and SmartRecruiters-hosted ATS pages continue to be JavaScript-rendered and returned only partial content to `WebFetch`. For those postings, records in `.aicg/job-requirements.json` carry a representative quote and null verbatim bullet fields with a note. The verbatim required/preferred bullets were only fully extracted for the Novara Software Built In posting this cycle (new) and the pre-existing fully-extracted postings carried from Oct (Phizenix, Anthropic Cyber, Cursor-prior, Hippocratic-prior, ServiceNow MLQE, Appnovation, OpenTrain, Future, Hippocratic Evals, WITHIN, Notion User Researcher, Novara, Block). Next cycle should try headless-browser rendering for the Ashby / Lever / Workday-native cohort.

## Methodology

1. Resampled 29 in-window postings (2026-08-05 → 2026-11-03) across the title cluster, spanning ATS aggregators (Greenhouse, Ashby, Lever, Wellfound, SmartRecruiters, Built In), the Workday-native board, and direct-employer career sites. 19 postings are retained from the Oct 2026 cycle's counted sample (same URL still live or indeterminate on 2026-11-03); 10 are newly surfaced.
2. For each posting, captured `employer / title / URL / date_observed / date_posted / location / verbatim required bullets / verbatim preferred bullets / salary_range / one representative quote` into [`.aicg/job-requirements.json → postings`](.aicg/job-requirements.json). For JS-rendered ATS pages where bullets were not extractable, the record kept the quote and marked bullet fields null with a note.
3. Attributed each posting to one or more `req-*` requirements via `requirement_evidence_for`.
4. For postings in Oct's 28-sample, re-verified the URL. Posts returning 404 / 410 / `removed on <date>` were marked expired via `status_note` on the posting record and excluded from current-window frequency math (while retained in the historical record). Posts whose body was ambiguous (board live but individual listing redirected / search snippet only) were kept conservatively in the current-window math.
5. Computed observed frequency per requirement as `distinct in-window counted postings citing / total in-window counted postings (29)`.
6. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. Deferred:
   - *Down* to `ml-engineer` (level 20) for classical ML fundamentals; to `llm-application-developer` / `rag-engineer` / `agentic-ai-engineer` (peer AI Engineering tracks) for the systems being evaluated.
   - *Sideways* to `model-evaluation-engineer` (peer, level 30, ML Engineering family) for statistical methodology depth, benchmark-construction depth, judge-vs-human calibration methodology depth, cross-modality model-eval methodology, and MLPerf-style serving benchmarks.
   - *Sideways* to `ai-evaluation-engineer` (peer, level 25, Governance family) for the release-assurance / audit-trail / regulator-facing shape.
   - *Sideways* to `ai-risk-engineer` (peer, level 30) for alignment-risk methodology, harm modelling, and red-team data generation.
   - *Up* to `staff-ml-engineer` / `principal-ml-engineer` / `senior-agentic-ai-engineer` / `agentic-systems-architect` / `head-of-ai-governance` / `ai-infra-security` for the leadership, architectural, and deep-security scopes that inherit this packet.
7. Applied the **continuity-bias rule** — a new module / exercise is only warranted when ≥3 distinct postings cite a requirement the existing curriculum does *not* cover AND observed frequency is ≥ 0.30 AND no existing module can be incrementally extended. No theme in this sample met all three conditions.

## Requirement themes → curriculum ownership

Frequency (`Freq`) is the share of the 29 current-window counted postings that cite the theme. Themes owned elsewhere are listed for routing purposes and have no observed frequency here.

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Product-shaped eval foundations: SLO mapping, offline vs. online split, deferring statistical depth | 0.38 | `ai-eval-engineer` (this) | [`mod-101-product-shaped-eval-foundations`](lessons/mod-101-product-shaped-eval-foundations) |
| 2 | Trace instrumentation for LLM apps and agents (OpenTelemetry / OpenInference, Phoenix / Langfuse / Weave / Braintrust / LangSmith) | 0.31 | `ai-eval-engineer` | [`mod-102-trace-instrumentation`](lessons/mod-102-trace-instrumentation) |
| 3 | Trajectory & tool-call eval for production agents (Inspect harness, SWE-bench Verified / τ-bench / WebArena / GAIA / BFCL / ToolBench shapes) | 0.52 | `ai-eval-engineer` | [`mod-103-trajectory-and-tool-eval`](lessons/mod-103-trajectory-and-tool-eval) |
| 4 | LLM-as-judge in product pipelines: rubric design, bias controls, judge-tier routing, judge-drift, quick calibration | 0.59 | `ai-eval-engineer` | [`mod-104-llm-as-judge-in-product`](lessons/mod-104-llm-as-judge-in-product) |
| 5 | RAG evaluation at the application layer (RAGAS / TruLens / DeepEval; RAG-triad; retrieval vs generation split) | 0.31 | `ai-eval-engineer` | [`mod-105-rag-eval-app-layer`](lessons/mod-105-rag-eval-app-layer) |
| 6 | Eval-gated CI/CD for prompts, chains, and agents (Promptfoo / DeepEval / Braintrust / Weave / Langfuse in CI) | 0.48 | `ai-eval-engineer` | [`mod-106-eval-gated-cicd`](lessons/mod-106-eval-gated-cicd) |
| 7 | Online evaluation & regression detection (sampled traces, drift, canary / shadow, sequential monitoring) | 0.34 | `ai-eval-engineer` | [`mod-107-online-eval-and-regression`](lessons/mod-107-online-eval-and-regression) |
| 8 | App-side safety & guardrails eval (jailbreak on surface, prompt-injection, tool abuse, guardrail effectiveness, OWASP LLM Top 10) | 0.34 | `ai-eval-engineer` | [`mod-108-app-safety-and-guardrails-eval`](lessons/mod-108-app-safety-and-guardrails-eval) |
| 9 | Human review workflows integrated with product feedback (Argilla / Label Studio, expert reviewers, product feedback loops) | 0.31 | `ai-eval-engineer` | [`mod-109-human-review-workflows`](lessons/mod-109-human-review-workflows) |
| 10 | Eval-data-platform slice: trace warehouse, dataset lineage, multi-runner orchestration, eval-as-a-service | 0.45 | `ai-eval-engineer` | [`mod-110-eval-data-platform-slice`](lessons/mod-110-eval-data-platform-slice) + [`project-103`](projects/project-103-ai-eval-platform-slice) |
| 11 | Cost / latency / quality trade-off eval (model routing, distillation regression, token / TTFT / TPOT at app altitude) | 0.03 | `ai-eval-engineer` | [`mod-111-cost-latency-quality-tradeoff`](lessons/mod-111-cost-latency-quality-tradeoff) |
| 12 | Owning an AI eval program: release-gate architecture, cross-team delegation, build-vs-buy, incident-driven investment | 0.41 | `ai-eval-engineer` | [`mod-112-owning-an-ai-eval-program`](lessons/mod-112-owning-an-ai-eval-program) |
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

- **Cost / latency / quality trade-off eval** — Firecrawl (still live), Anthropic Beneficial Deployments. 1 explicit new mention; Cursor (expired) dropped out. Explicit frequency 0.03. Already the exact scope of `mod-111-cost-latency-quality-tradeoff`. The theme is load-bearing in build-vs-buy discussions and shows up implicitly (Rubrik's quad-dimension `correctness / latency / cost / safety` framing is the clearest implicit mention this cycle) in most senior postings — do not demote.
- **MCP server / tool-registry eval** — Workday Senior AI Evaluation Engineer explicit (first mainstream app-altitude mention in bullets). 1/29 = 0.03. Below both gates and already subsumed by `mod-103` tool-call eval. Watch next cycle as MCP adoption grows.
- **Long-horizon / multi-turn trajectory eval** — Workday, ServiceNow Moveworks, Rubrik. 3/29 = 0.10. Already inside `mod-103` under deterministic replay + agent-harness authoring. If frequency crosses 0.30 next cycle, extend `mod-103` with a long-horizon exercise before considering a new module.
- **Evals-driven synthetic data generation at app altitude** — Workday explicit (`synthetic multi-turn trajectory ... eval suites for release gates`), Netomi (`synthetic-conversation data pipelines`), Judi Health (nice-to-have voice/call-center synthetic data), Cortea (implicit). 3–4/29 = 0.10–0.14. Meets the 3-posting evidence bar but FAILS the ≥0.30 frequency gate. Already partially covered by `mod-109-human-review-workflows` (dataset authoring + labelling) and `mod-110-eval-data-platform-slice` (dataset lineage). First theme to re-check next cycle; if it crosses 0.30, extend `mod-109` with a targeted exercise before considering a new module.
- **Multimodal / voice-agent eval at app altitude** — Judi Health (voice / call-center transcripts, nice-to-have), Apple SDET (image / visual generation judging). 2/29 = 0.07. Owned at the model-eval altitude by `model-evaluation-engineer` and at app altitude in Voice & Multimodal Agent Engineer (`voice-agent-engineer-learning`, level 35) when the surface is explicitly voice/multimodal. This track evaluates text-shaped surfaces.
- **Cyber-evaluation of AI** — Anthropic Cyber Evaluations Engineer (still live). 1/29 in-window. Specialist adjacent to `mod-108`; the *deep* security shape is owned upstream by `ai-infra-security-learning` (level 35). Do not spin out.
- **Simulated-user / rollout environments** — Hippocratic AI (expired), Precept Labs (expired), Cursor (expired), Anomali (expired). Theme is dropping out as the Oct cohort of simulated-user postings expires; current-window count only 1 (Rubrik's agent-harness end-to-end). Already inside `mod-103` under deterministic replay + agent-harness authoring. Do not spin out.
- **Coding-agent-specific eval** — OpenTrain AI (still live), Phizenix (still live), METR (still live, routed sideways). ~3/29 explicit. Already covered by `mod-103`'s SWE-bench Verified / BFCL / τ-bench shape templates and the tool-call scoring exercise.
- **Domain-vertical eval templates (health, legal, fintech, life-sciences, cybersecurity, dealer-automotive, finance/audit)** — Hippocratic AI (expired), Phizenix, Appnovation, Distyl, Notion, NCS, Tekion, Anomali (expired), Onit, Rubrik, Cortea, Judi Health. Domain vertical is the *context* of the eval, not a distinct requirement — the underlying skills are already covered by the existing modules and the projects already ask learners to instantiate them on a specific product surface.
- **Reasoning-model / chain-of-thought-faithfulness eval** — 0/29 explicit in product-eng postings this cycle. Research-stage topic. Owned at the model-eval altitude by `model-evaluation-engineer`.
- **Agent-memory eval** — 0/29 explicit. Watch next cycle.
- **Evaluator-model / judge-model fine-tuning** — 0/29 explicit in app-altitude postings. Partially inside `mod-104` (judge calibration + judge-drift).

## Posting evidence

The 29 in-window counted postings this cycle (2026-08-05 → 2026-11-03). Rows 1–19 are the still-live-or-indeterminate cohort retained from the 2026-10 cycle's 28-sample; rows 20–29 are the 10 newly surfaced postings. See [`.aicg/job-requirements.json → postings`](.aicg/job-requirements.json) for the full verbatim record, including the 9 postings from the 2026-10 cycle (Cursor, Hippocratic AI builtin, GovWorx, Distyl B, OpenAI Backend Evals, Precept Labs, Hippocratic Senior MLE, Anomali, Block) that were confirmed expired on 2026-11-03 and the 6 postings from earlier cycles (WITHIN, Distyl A, Notion User Researcher, White Circle, HackerRank, Atlassian) that remain in the historical record only.

| # | Employer | Title | Salary | Requirements evidence for |
|---|---|---|---|---|
| 1 | Phizenix | [LLM / Agentic Evaluation Rig Engineer](https://job-boards.greenhouse.io/phizenix/jobs/5398766008) | — | mod-101, mod-103, mod-104, mod-105, mod-106, mod-107 |
| 2 | Anthropic | [Cyber Evaluations Engineer](https://job-boards.greenhouse.io/anthropic/jobs/5406367008) | $300k–$405k | mod-108, mod-112 |
| 3 | Glean | [Software Engineer, Evals](https://job-boards.greenhouse.io/gleanwork/jobs/4712438005) | — | mod-102, mod-103, mod-107, mod-110, mod-112 |
| 4 | ServiceNow | [Machine Learning Quality Engineer](https://builtin.com/job/machine-learning-quality-engineer/6607682) | $124k–$192k | mod-101, mod-104, mod-106, mod-108 |
| 5 | Appnovation Technologies | [AI Evaluation Engineer (QA)](https://job-boards.greenhouse.io/appnovation/jobs/8743206002) | — | mod-101, mod-105, mod-106, mod-109, mod-110 |
| 6 | OpenTrain AI | [Python Engineer, AI Coding Agent Evaluation](https://www.opentrain.ai/jobs/python-engineer-ai-coding-agent-evaluation--cmtfopkq0000j0akoeodsq186/) | — | mod-103, mod-109 |
| 7 | Future | [Applied AI Engineer](https://job-boards.greenhouse.io/future/jobs/4683133005) | $215k–$250k | mod-102, mod-104, mod-106, mod-110 |
| 8 | CI&T | [AI Agent Evaluation Engineer (Senior, QA)](https://jobs.lever.co/ciandt/1e06dadb-5342-470d-a9a1-cb42755381a2) | — | mod-102, mod-103, mod-104, mod-105, mod-109 |
| 9 | Coupa Software | [AI Engineer, Evaluation & Quality](https://jobs.lever.co/coupa/0e7ed42f-a206-4ea6-afe6-4e312b1f7046) | — | mod-106, mod-107, mod-110 |
| 10 | Nous Research | [Machine Learning Engineer, Evals](https://jobs.ashbyhq.com/nous-research/1e647dc1-e69c-4764-8a02-7244b8faee0b) | — | mod-104, mod-110 |
| 11 | Exa | [ML Evals Engineer](https://jobs.ashbyhq.com/exa/9c45a74e-d507-482a-bc0e-da2f464c9767) | — | mod-105, mod-110 |
| 12 | Thinking Machines Lab | [Software Engineer, Evaluation Platform / Infra](https://jobs.ashbyhq.com/thinkingmachines/9d863c78-80c0-44cd-a574-d1330e125398) | — | mod-110, mod-112 |
| 13 | White Circle | [Research Engineer (Evals)](https://jobs.ashbyhq.com/whitecircle/07d24ec8-117f-4849-8dd5-2c2bc2cfbb40) | — | mod-103, mod-108 |
| 14 | Firecrawl | [Research Engineer – Evals](https://jobs.ashbyhq.com/firecrawl/25092c0e-9a32-4191-af79-050738213704) | $160k–$240k | mod-101, mod-106, mod-110, mod-112 |
| 15 | NCS | [LLM / AI Quality Engineer](https://jobs.smartrecruiters.com/NCS3/6000000000955545--eg-llm-ai-quality-engineer) | — | mod-101, mod-103, mod-105, mod-108, mod-112 |
| 16 | Tekion | [Staff SDET (GenAI & LLM Evaluation)](https://jobs.ashbyhq.com/Tekion/c7a28d8f-dee9-47c0-a314-93c61c590244) | — | mod-104, mod-106, mod-108 |
| 17 | Onit | [AI Engineer (Evaluation Frameworks)](https://jobs.lever.co/onit/983d3c5d-55d7-4802-9865-80bf58c68bd4) | — | mod-101, mod-106 |
| 18 | Netomi | [Staff Agentic AI Engineer](https://jobs.lever.co/netomi/3fe31ab4-1e79-4b1c-8493-dfec68e69ba7) | — | mod-103, mod-104, mod-106, mod-107, mod-108 |
| 19 | METR | [Task Development Engineer](https://jobs.lever.co/metr/b4812bf4-c259-406b-8ffa-4a463fff34f7) | $260.9k–$385.5k | mod-103 (frontier-adjacent; depth routed to peer `model-evaluation-engineer`) |
| 20 | Novara Software | [Senior AI Quality Engineer](https://builtin.com/job/senior-ai-quality-engineer/11406388) | $100k–$130k | mod-101, mod-103, mod-104, mod-106, mod-108 |
| 21 | Workday | [Senior AI Evaluation Engineer (CI/CD & Platform)](https://www.dreamworkhq.com/job/9a2a60e9-d27c-4a81-8873-cc84966f5ce8) | €96k–€144k | mod-101, mod-102, mod-103, mod-104, mod-105, mod-107, mod-109, mod-110, mod-112 (watch-list: MCP, synthetic multi-turn trajectory) |
| 22 | ServiceNow (Moveworks) | [Staff Software Engineer, Agent Eval Platform](https://careers.servicenow.com/jobs/744000144602090/staff-software-engineer-agent-eval-platform/) | — | mod-102, mod-103, mod-104, mod-106, mod-108, mod-112 (watch-list: long-horizon stateful trajectory) |
| 23 | Cortea AI | [Senior / Staff AI Engineer, Quality & Evals](https://www.arbeitnow.com/jobs/companies/cortea-ai/senior-staff-ai-engineer-quality-evals-berlin-387619) | — | mod-101, mod-102, mod-103, mod-104, mod-107, mod-109, mod-110 (domain: finance / audit) |
| 24 | Apple | [Senior SDET, LLM Evaluation & Automation (T3E)](https://jobs.anitab.org/companies/apple/jobs/88993879-senior-software-development-engineer-in-test-llm-evaluation-automation-t3e) | $175k–$308.5k | mod-103, mod-104, mod-105, mod-108, mod-109, mod-110 (watch-list: multimodal image/visual judge) |
| 25 | Judi Health | [AI Evaluation Engineer](https://jobs.generalcatalyst.com/companies/judi-health-2/jobs/86572762-ai-evaluation-engineer) | $134.8k–$200k | mod-101, mod-102, mod-103, mod-104, mod-107, mod-109, mod-110, mod-112 (watch-list: voice/call-center transcripts) |
| 26 | Docket | [AI Evals Engineer (LLM Evaluation & Customer Traceability)](https://jobs.foundationcapital.com/companies/docket-2-74a3481d-f23b-4f38-921c-1fcd5d7cb139/jobs/57245072-ai-evals-engineer-llm-evaluation-customer-traceability) | — | mod-101, mod-102, mod-103, mod-104, mod-105, mod-107, mod-109 (date_posted uncertain; low-confidence inclusion) |
| 27 | Mistral | [Evaluation Engineer, Applied AI](https://jobs.ashbyhq.com/mistral.ai/5e6f3efd-36a5-48ab-8643-843edaeee578) | — | mod-104, mod-106, mod-112 (low-confidence: JS-rendered body) |
| 28 | Rubrik | [Senior AI Evals Engineer](https://www.rubrik.com/company/careers/departments/job.8273834) | ₹42L–₹65L INR | mod-102, mod-103, mod-104, mod-105, mod-106, mod-107, mod-108, mod-109, mod-112 (domain: cybersecurity; implicit mod-111 via quad-dimension framing) |
| 29 | Anthropic | [Applied AI Engineer, Beneficial Deployments (Lab Automation)](https://job-boards.greenhouse.io/anthropic/jobs/5440821008) | £225k–£255k | mod-104, mod-106, mod-108, mod-111, mod-112 (niche: lab automation deployment) |
| 30 | Datadog | [Engineering Manager I, AI Platform – Evaluation & Annotation](https://www.opentalent.in/jobs/engineering-manager-i-ai-platform-evaluation-and-annotation-datadog) | — | mod-107, mod-110, mod-112 (edge of window; EM-shaped role) |

> Rows numbered 1–30 by display order; 29 counted postings (row 30 = Datadog is counted as new even though date_posted 2026-07-01 is one month before window start, because the posting is still active on 2026-11-03 — explicitly flagged in the record).

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
- **Voice & Multimodal Agent Engineer** (level 35) — if the eval surface is explicitly voice or multimodal (not text), the specialist eval depth is owned there. mod-103, mod-104, and mod-108 cover the shared app-altitude evaluation vocabulary.

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
