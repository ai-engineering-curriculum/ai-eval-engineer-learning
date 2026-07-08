# mod-108-app-safety-and-guardrails-eval: App-Side Safety and Guardrail-Effectiveness Evaluation

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Measure jailbreak-resistance on the *actual application surface* (not the raw model) using HarmBench-style attack suites without leaking harmful payloads into the repo
- Measure prompt-injection robustness in RAG and tool-calling agents using AgentDojo / InjecAgent and derived internal suites
- Measure tool-abuse / unintended-tool-use (data exfiltration via retrieval, out-of-scope tool invocation) with sandboxed replay
- Measure guardrail effectiveness (NeMo Guardrails, Guardrails AI, Llama Guard, provider moderation) with explicit false-positive / false-negative reporting and cost accounting
- Measure refusal / over-refusal at the app surface and reconcile with the safety policy
- Map findings to OWASP Top 10 for LLM Applications, MITRE ATLAS, Google SAIF and know the delegation boundary to `ai-risk-engineer` (harm-model / red-team data) and to `model-evaluation-engineer` (dangerous-capability model-level evals)

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
