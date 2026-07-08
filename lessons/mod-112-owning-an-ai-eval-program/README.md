# mod-112-owning-an-ai-eval-program: Owning an AI Evaluation Program Across Product, Infra, Safety, and Governance

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 12 hours

## Learning objectives

- Translate a product roadmap into a release-gate architecture for one or more AI application surfaces, with explicit thresholds and rollback criteria
- Author delegation contracts to peer specialists: `model-evaluation-engineer` (model-eval methodology depth), `ai-risk-engineer` (harm models & red-team data), `ai-evaluation-engineer` / `ai-governance-analyst` (release-assurance & regulator interface), `ai-infra-security` (adversarial-eval & judge supply-chain), `ai-infra-mlops` / `ai-infra-ml-platform` (CI and platform integration)
- Make the build-vs-buy decision across app-eval platforms (Weave, Phoenix, Langfuse, Braintrust, Promptfoo, DeepEval, RAGAS, Humanloop, Galileo, Patronus, Vertex Eval, Bedrock Eval, Azure AI Foundry Eval) with cost, data-residency, extensibility, and vendor-lock trade-offs explicit
- Run incident-driven eval investment: turn a real production incident into a permanent regression fixture, an alert, and a runbook update
- Compose a product-side model / system card slice from the eval evidence that satisfies internal review and hands cleanly to the governance-family peer
- Map the program to NIST AI RMF Measure/Manage, EU AI Act general-purpose AI model provider obligations, and ISO/IEC 42001 controls — knowing that governance shape is owned by `ai-evaluation-engineer` (peer, Governance family, level 25) and by `ai-governance-analyst`

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
