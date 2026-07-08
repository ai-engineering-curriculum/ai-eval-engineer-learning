# Job Requirements — AI Evaluation Engineer

**Role level:** 30 (deep specialist, AI Engineering family — peer to `model-evaluation-engineer` on the ML Engineering ladder and to `senior-ml-engineer` at the same level)
**Track:** `ai-eval-engineer-learning`
**Research window:** 2026-04-09 → 2026-07-08 (last 90 days)
**Today:** 2026-07-08

This file documents the requirements catalog used to seed the AI Evaluation Engineer curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json).

## Status — bootstrap session, postings deferred

<!-- needs-research: collect ≥25 distinct in-window postings titled "AI Evaluation Engineer" / "LLM Evaluation Engineer" / "GenAI Evaluation Engineer" / "AI Quality Engineer" / "AI Test Engineer" / "Applied AI Evaluation Engineer" / "AI Application Evaluation Engineer" / "Agentic AI Evaluation Engineer" / "AI Product Evaluation Engineer" / "Evaluation Engineer (LLM / GenAI)". Filter OUT "Senior / Staff / Principal" modifiers (those inherit this packet), generic "ML Engineer / Data Scientist / Research Scientist" titles (owned by ml-engineer / senior-ml-engineer / research-scientist peers), and postings whose emphasis is benchmark-construction methodology, statistical rigour, or cross-modality model-eval architecture (those belong to `model-evaluation-engineer`, the peer level-30 specialist in the ML Engineering family). Re-validate every requirement against the live evidence and demote any whose evidence stays empty. -->

This packet was authored in a bootstrap session **without an exercised WebSearch / WebFetch pass against live job boards** (WebSearch permission was not granted in-session). Per the project rules (*"Do not invent facts, incidents, or salary figures. Cite sources."*), the `postings` array in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) is intentionally empty — the curriculum will not claim to have analysed 25 live postings when none were fetched.

The autonomous research loop is expected to fill that gap on its next cycle. To keep the loop deterministic and to give the curriculum a defensible baseline in the meantime, this document grounds the requirements catalog in **authoritative public references** that publish what the role is hired against:

- **Application-layer LLM/agent eval frameworks and platforms** — [Arize Phoenix](https://docs.arize.com/phoenix), [Langfuse](https://langfuse.com/docs), [W&B Weave](https://weave-docs.wandb.ai/), [Braintrust](https://www.braintrust.dev/docs/start), [LangSmith](https://docs.smith.langchain.com/), [Promptfoo](https://www.promptfoo.dev/docs/intro/), [DeepEval](https://docs.confident-ai.com/), [RAGAS](https://docs.ragas.io/), [TruLens](https://www.trulens.org/), [Humanloop](https://humanloop.com/docs), [Galileo](https://docs.galileo.ai/), [Patronus](https://docs.patronus.ai/), and the AISI [Inspect](https://inspect.aisi.org.uk/) agent harness — these are the concrete tools the role is expected to wield in postings.
- **Observability conventions** — [OpenTelemetry Gen-AI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/), [OpenInference span attributes](https://github.com/Arize-ai/openinference).
- **Agent evaluation research** — MT-Bench / Chatbot Arena ([arXiv:2306.05685](https://arxiv.org/abs/2306.05685)), G-Eval ([arXiv:2303.16634](https://arxiv.org/abs/2303.16634)), Prometheus 2 ([arXiv:2405.01535](https://arxiv.org/abs/2405.01535)), SWE-bench ([arXiv:2310.06770](https://arxiv.org/abs/2310.06770)) + [SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/), WebArena ([arXiv:2307.13854](https://arxiv.org/abs/2307.13854)), τ-bench ([arXiv:2406.12045](https://arxiv.org/abs/2406.12045)), ToolBench ([arXiv:2307.16789](https://arxiv.org/abs/2307.16789)), [Berkeley Function-Calling Leaderboard](https://gorilla.cs.berkeley.edu/leaderboard.html), AgentBench ([arXiv:2308.03688](https://arxiv.org/abs/2308.03688)), GAIA ([arXiv:2311.12983](https://arxiv.org/abs/2311.12983)), RAGAS ([arXiv:2309.15217](https://arxiv.org/abs/2309.15217)).
- **App-side safety and prompt-injection research** — HarmBench ([arXiv:2402.04249](https://arxiv.org/abs/2402.04249)), AgentDojo ([arXiv:2406.13352](https://arxiv.org/abs/2406.13352)), InjecAgent ([arXiv:2403.02691](https://arxiv.org/abs/2403.02691)), AIR-Bench 2024 ([arXiv:2407.17436](https://arxiv.org/abs/2407.17436)), SafetyBench ([arXiv:2309.07045](https://arxiv.org/abs/2309.07045)), Llama Guard ([arXiv:2312.06674](https://arxiv.org/abs/2312.06674)), GCG ([arXiv:2307.15043](https://arxiv.org/abs/2307.15043)).
- **App-side guardrails and moderation** — [NVIDIA NeMo Guardrails](https://docs.nvidia.com/nemo/guardrails/), [Guardrails AI](https://www.guardrailsai.com/docs), [Meta Llama Guard / Purple Llama](https://ai.meta.com/research/publications/llama-guard-llm-based-input-output-safeguard-for-human-ai-conversations/), [OpenAI Moderation](https://platform.openai.com/docs/guides/moderation), [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/), [MITRE ATLAS](https://atlas.mitre.org/), [Google SAIF](https://safety.google/cybersecurity-advancements/saif/).
- **Statistical methodology (link-out, not depth)** — Efron ([1979 bootstrap](https://projecteuclid.org/journals/annals-of-statistics/volume-7/issue-1/Bootstrap-Methods-Another-Look-at-the-Jackknife/10.1214/aos/1176344552.full)), Benjamini-Hochberg ([1995 FDR](https://www.jstor.org/stable/2346101)), Deng et al. ([2013 CUPED](https://www.exp-platform.com/Documents/2013-02-CUPED-ImprovingSensitivityOfControlledExperiments.pdf)), Howard et al. ([2018 confidence sequences](https://arxiv.org/abs/1810.08240)), [Kohavi/Tang/Xu (2020 Trustworthy Online Controlled Experiments)](https://www.cambridge.org/9781108724265).
- **Governance / release-gate references (link-out)** — [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework), [NIST AI RMF Generative AI Profile (AI 600-1)](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile), [EU AI Act (Regulation 2024/1689)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj), [ISO/IEC 42001:2023](https://www.iso.org/standard/81230.html), [ISO/IEC 25059](https://www.iso.org/standard/80655.html), [Mitchell et al. Model Cards](https://arxiv.org/abs/1810.03993), public OpenAI system cards, public Anthropic Claude model card disclosures.
- **Managed-eval build-vs-buy references** — [Vertex AI Gen AI Evaluation](https://cloud.google.com/vertex-ai/generative-ai/docs/models/evaluation-overview), [AWS Bedrock Model Evaluation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-evaluation.html), [Azure AI Foundry Evaluation SDK](https://learn.microsoft.com/en-us/azure/ai-studio/how-to/develop/evaluate-sdk).

Every requirement below cites at least one such reference, and every requirement is shaped so that posting-frequency evidence can be added underneath it without restructure.

## Methodology

1. Sourced the canonical task domains for a deep-specialist AI-application evaluation engineer from the public references catalogued in `.aicg/job-requirements.json → authoritative_references`.
2. Mapped each task domain to (a) the role on our level ladder that should own it primarily and (b) the curriculum module that will cover it.
3. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. In particular, deferred:
   - *Down* to `ml-engineer` (level 20) for classical ML fundamentals; to `llm-application-developer` / `rag-engineer` / `agentic-ai-engineer` (peer AI Engineering tracks) for the systems being evaluated.
   - *Sideways* to `model-evaluation-engineer` (peer, level 30, ML Engineering family) for statistical methodology depth, benchmark-construction depth, judge-vs-human calibration methodology depth, cross-modality model-eval methodology, and MLPerf-style serving benchmarks.
   - *Sideways* to `ai-evaluation-engineer` (peer, level 25, Governance family) for the release-assurance / audit-trail / regulator-facing shape.
   - *Sideways* to `ai-risk-engineer` (peer, level 30) for alignment-risk methodology, harm modelling, and red-team data generation.
   - *Up* to `staff-ml-engineer` / `principal-ml-engineer` / `senior-agentic-ai-engineer` / `agentic-systems-architect` / `head-of-ai-governance` / `ai-infra-security` for the leadership, architectural, and deep-security scopes that inherit this packet.
4. Flagged everything that has not yet been validated against in-window postings so the next cycle can demote any requirement whose evidence stays empty.

## Requirement themes → curriculum ownership

The table below lists each requirement theme, its planned owner per the level hierarchy, and the curriculum coverage path. **Freq** is intentionally blank for this cycle — it will be backfilled by the next research pass.

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Product-shaped eval foundations: SLO mapping, offline vs. online split, deferring statistical depth | <!-- needs-research --> | `ai-eval-engineer` (this) | [`mod-101-product-shaped-eval-foundations`](lessons/mod-101-product-shaped-eval-foundations) |
| 2 | Trace instrumentation for LLM apps and agents (OpenTelemetry / OpenInference, Phoenix / Langfuse / Weave / Braintrust / LangSmith) | <!-- needs-research --> | `ai-eval-engineer` | [`mod-102-trace-instrumentation`](lessons/mod-102-trace-instrumentation) |
| 3 | Trajectory & tool-call eval for production agents (Inspect harness, SWE-bench Verified / τ-bench / WebArena / GAIA / BFCL / ToolBench shapes) | <!-- needs-research --> | `ai-eval-engineer` | [`mod-103-trajectory-and-tool-eval`](lessons/mod-103-trajectory-and-tool-eval) |
| 4 | LLM-as-judge in product pipelines: rubric design, bias controls, judge-tier routing, judge-drift, quick calibration | <!-- needs-research --> | `ai-eval-engineer` | [`mod-104-llm-as-judge-in-product`](lessons/mod-104-llm-as-judge-in-product) |
| 5 | RAG evaluation at the application layer (RAGAS / TruLens / DeepEval; RAG-triad; retrieval vs generation split) | <!-- needs-research --> | `ai-eval-engineer` | [`mod-105-rag-eval-app-layer`](lessons/mod-105-rag-eval-app-layer) |
| 6 | Eval-gated CI/CD for prompts, chains, and agents (Promptfoo / DeepEval / Braintrust / Weave / Langfuse in CI) | <!-- needs-research --> | `ai-eval-engineer` | [`mod-106-eval-gated-cicd`](lessons/mod-106-eval-gated-cicd) |
| 7 | Online evaluation & regression detection (sampled traces, drift, canary / shadow, sequential monitoring) | <!-- needs-research --> | `ai-eval-engineer` | [`mod-107-online-eval-and-regression`](lessons/mod-107-online-eval-and-regression) |
| 8 | App-side safety & guardrails eval (jailbreak on surface, prompt-injection, tool abuse, guardrail effectiveness, OWASP LLM Top 10) | <!-- needs-research --> | `ai-eval-engineer` | [`mod-108-app-safety-and-guardrails-eval`](lessons/mod-108-app-safety-and-guardrails-eval) |
| 9 | Human review workflows integrated with product feedback (Argilla / Label Studio, expert reviewers, product feedback loops) | <!-- needs-research --> | `ai-eval-engineer` | [`mod-109-human-review-workflows`](lessons/mod-109-human-review-workflows) |
| 10 | Eval-data-platform slice: trace warehouse, dataset lineage, multi-runner orchestration, eval-as-a-service | <!-- needs-research --> | `ai-eval-engineer` | [`mod-110-eval-data-platform-slice`](lessons/mod-110-eval-data-platform-slice) + [`project-103`](projects/project-103-ai-eval-platform-slice) |
| 11 | Cost / latency / quality trade-off eval (model routing, distillation regression, token / TTFT / TPOT at app altitude) | <!-- needs-research --> | `ai-eval-engineer` | [`mod-111-cost-latency-quality-tradeoff`](lessons/mod-111-cost-latency-quality-tradeoff) |
| 12 | Owning an AI eval program: release-gate architecture, cross-team delegation, build-vs-buy, incident-driven investment | <!-- needs-research --> | `ai-eval-engineer` | [`mod-112-owning-an-ai-eval-program`](lessons/mod-112-owning-an-ai-eval-program) |
| 13 | Classical ML / PyTorch / sklearn-eval / packaging fundamentals | n/a — prerequisite | `ml-engineer` (level 20) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught |
| 14 | LLM application development (prompting, tool design, chain orchestration) | n/a — peer track | `llm-application-developer` (level 25) | Linked out; this curriculum evaluates what that role builds |
| 15 | RAG system engineering (chunking, embeddings, rerankers) | n/a — peer track | `rag-engineer` (level 25) | mod-105 covers RAG *eval*; RAG engineering linked out |
| 16 | Agent implementation (ReAct, LangGraph / CrewAI / AutoGen, MCP) | n/a — peer track | `agentic-ai-engineer` (level 30) | mod-103 evaluates agents; implementation linked out |
| 17 | Model-eval methodology depth: validity theory, benchmark construction, statistical estimation, judge-vs-human calibration, MLPerf | n/a — peer specialist | `model-evaluation-engineer` (level 30) | Consumer contract in mod-101 and mod-104; depth linked out |
| 18 | Release-assurance / regulator-facing shape (audit trails, third-party evaluator interface) | n/a — peer track | `ai-evaluation-engineer` (level 25, Governance family) | mod-112 documents the interface; assurance shape linked out |
| 19 | Alignment-risk methodology / harm modelling / red-team data generation | n/a — peer specialist | `ai-risk-engineer` (level 30) | mod-108 measures; harm-model design linked out |
| 20 | Post-training (SFT / PEFT / RLHF / DPO) depth | n/a — peer specialist | `fine-tuning-engineer` (level 30) | Out of scope — this curriculum evaluates the outputs |
| 21 | Distributed training PLATFORM engineering | n/a — peer track | `training-pipeline-engineer` (level 25) | Out of scope |
| 22 | Deep ML/AI security (model extraction, eval-set exfiltration, judge supply-chain attacks) | n/a — higher level | `ai-infra-security-learning` (level 35) | Awareness in mod-108 and mod-110; depth owned upstream |
| 23 | Compliance / model-card review / regulated-data review depth | n/a — peer track | `ai-governance-analyst` (level 25) | Awareness in mod-112; depth owned upstream |

## Posting evidence

<!-- needs-research: populate with the ≥25 in-window postings sampled next cycle. Use the same table shape as `agentic-ai-engineer-learning/JOB_REQUIREMENTS.md`. -->

No postings were sampled this cycle. See the **Status** section above for the reason. The next autonomous research cycle should fan out across:

- **ATS aggregators** — `boards.greenhouse.io`, `jobs.lever.co`, `jobs.ashbyhq.com`, `app.workable.com`, `myworkdayjobs.com`, `smartrecruiters.com`.
- **Frontier-lab AI-application teams** — OpenAI (Applied AI / product eval), Anthropic (Applied AI / product eval), Google DeepMind (product eval), Meta GenAI, Microsoft AI, Apple AIML, Amazon AGI, Salesforce AI, NVIDIA (product / applied), xAI, Reka, Character.AI. Reject pre-training / capability-eval roles at these employers — those belong to `model-evaluation-engineer`.
- **Eval-tooling employers** — Braintrust, Arize AI (Phoenix), Weights & Biases (Weave), Langfuse, Confident-AI (DeepEval), Promptfoo, Patronus, Galileo, Humanloop, LangChain (LangSmith), LlamaIndex, TruLens, Snorkel, Argilla, Cleanlab, Lakera.
- **Applied AI product companies** — Notion AI, Duolingo AI, Klarna, Perplexity, Harvey, Sierra, Glean, Hebbia, Cursor, Codeium, Windsurf, Cognition, Adept, Character.AI, Inflection, You.com.
- **Hyperscaler managed-eval product teams** — Vertex AI Gen AI Evaluation (Google), Azure AI Foundry Evaluation (Microsoft), AWS Bedrock Model Evaluation.
- **Enterprise AI teams** — JPMorgan AI, Bloomberg AI, Bank of America, ServiceNow AI, Adobe Sensei / Firefly, Salesforce Einstein Trust.
- **Domain-specialist applied-AI teams** — Included Health, Ambience, Nuance / DAX (Microsoft), Robin AI, EvenUp, Casetext (Thomson Reuters), Harvey, Osmo, Genesys AI, Zendesk AI.
- **Public-sector adjacent** — UK AI Safety Institute product evaluations, US AI Safety Institute applied evaluations (only when the posting is app-shaped, not model-shaped).

For each posting, capture employer, exact title, URL, `date_observed`, `date_posted` (or `estimated:2026-MM`), location, 5–10 verbatim requirement bullets, 2–6 preferred-qualification bullets, salary range when published, and one short representative quote. Filter out:

- **Senior / Staff / Principal modifiers** — those inherit this packet upward.
- **Model-eval methodology postings** (benchmark construction, statistical rigour, multi-modality model-eval architecture) — those belong to `model-evaluation-engineer` at the same level.
- **Governance-shaped postings** (audit trails, regulator interface, release governance as an assurance program) — those belong to `ai-evaluation-engineer` (level 25, Governance family).
- **Generic ML Engineer / Data Scientist / Research Scientist titles** — those are owned by `ml-engineer`, `senior-ml-engineer`, and research-scientist peers.
- **Pure infra / platform titles** — those belong to `training-pipeline-engineer` and `ai-infra-ml-platform`.

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
- **AI Infra Security** (level 35) — owns deep AI/ML security (eval-set exfiltration, judge supply-chain, adversarial-eval depth); surfaced as awareness only.

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
