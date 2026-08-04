# mod-108 — App-Side Safety and Guardrail-Effectiveness Evaluation

**Estimated effort:** 14 hours

This is the eighth module of the AI Evaluation Engineer track. It builds the *safety half* of the eval program — the surface-level jailbreak resistance, prompt-injection robustness, tool-abuse containment, guardrail effectiveness, and refusal-policy monitoring that a shipped LLM application has to defend continuously.

Mod-107 built the online-eval loop against quality, cost, and latency; the same loop with stricter thresholds and zero-tolerance safety floors is what this module reuses. What is new is *what gets scored*: attack suites shaped like HarmBench, prompt-injection benchmarks shaped like AgentDojo and InjecAgent, sandboxed tool-abuse replays, guardrail confusion-matrix reports, and refusal / over-refusal reconciliations against a written safety policy.

The module deliberately does not re-derive the harm model or generate the raw red-team data (owned by the peer `ai-risk-engineer`), and it does not run dangerous-capability model-level evaluations (owned by `model-evaluation-engineer`). It walks the *application-surface* mechanism — the attack replay, the injection suite, the sandbox harness, the guardrail scorecard, the refusal calibration — and names the interfaces where the peers take over.

## Learning objectives

- Measure jailbreak resistance on the *actual application surface* (not the raw model) using HarmBench-style attack suites without leaking harmful payloads into the repo.
- Measure prompt-injection robustness in RAG and tool-calling agents using AgentDojo, InjecAgent, and derived internal suites.
- Measure tool-abuse and unintended-tool-use (data exfiltration via retrieval, out-of-scope tool invocation) with a sandboxed replay harness.
- Measure guardrail effectiveness (NeMo Guardrails, Guardrails AI, Llama Guard, provider moderation) with explicit false-positive and false-negative reporting plus cost accounting.
- Measure refusal and over-refusal at the app surface and reconcile with the written safety policy.
- Map findings to OWASP Top 10 for LLM Applications, MITRE ATLAS, and Google SAIF, and know the delegation boundary to `ai-risk-engineer` (harm model, red-team data) and `model-evaluation-engineer` (dangerous-capability model-level evals).

## Lecture chapters

1. [Why app-side safety eval, and where it hands off](01-why-app-side-safety-eval.md) — the model-level / app-level split; the five safety-eval axes this module covers (jailbreak resistance, injection robustness, tool-abuse containment, guardrail effectiveness, refusal reconciliation); the eval-team / risk-team / model-eval-team responsibility triangle; six vocabulary terms the module reuses.
2. [Jailbreak resistance at the app surface (HarmBench shape)](02-jailbreak-resistance-harmbench-shape.md) — HarmBench's behaviour × attack matrix; ASR / ASR@k / ASR-per-attack-family as metrics; the payload-management contract (harmful payloads never in the repo; hashes, references, and sealed archives instead); running against the *application surface* rather than the raw model; the interaction with the mod-107 online loop.
3. [Prompt-injection robustness for RAG and tool-calling agents](03-prompt-injection-agentdojo-injecagent.md) — the direct / indirect / mixed injection taxonomy; AgentDojo's user-goal / injection-goal split and its `utility × security` scorecard; InjecAgent's tool-abuse focus; injection channels (retrieved documents, tool outputs, memory, upstream model outputs); building derived internal suites without inventing scary payloads.
4. [Tool abuse and exfiltration with sandboxed replay](04-tool-abuse-and-exfiltration-sandbox.md) — out-of-scope tool invocation; data exfiltration via retrieval and via tool arguments; canary tokens in sandbox documents; the sealed sandbox pattern (network egress denied, side effects captured, replays deterministic); the exfiltration-rate scorecard.
5. [Guardrail effectiveness with an explicit FP / FN report](05-guardrail-effectiveness-fp-fn.md) — the guardrail stack survey (NeMo Guardrails, Guardrails AI, Llama Guard, OpenAI Moderation, Azure AI Content Safety, Google Model Armor); the confusion-matrix scorecard; ROC / PR curves and the operating point; the cost-per-guarded-request line item; the latency budget for a guardrail on the critical path.
6. [Refusal, over-refusal, and the safety-policy reconciliation](06-refusal-over-refusal-policy.md) — measuring refusal at the app surface, not the raw model; XSTest and OR-Bench-style suites for over-refusal; the written safety-policy artefact and the row-by-row reconciliation; the "helpful, harmless, honest" trilemma at product scale; talking to product about the trade.
7. [OWASP LLM Top 10, MITRE ATLAS, SAIF, and the delegation boundary](07-owasp-atlas-saif-delegation.md) — mapping findings to OWASP LLM Top 10 (2025), MITRE ATLAS tactics and techniques, Google SAIF's six risk-mitigation categories, and NIST AI RMF GenAI Profile; the safety-eval runbook shape; where the report ends and the risk-team / model-eval-team investigations begin.

## Exercises

Each exercise builds on the last. Do them in order and keep the outputs — the artefacts (jailbreak scorecard, injection scorecard, sandboxed tool-abuse harness, guardrail FP / FN report, OWASP-mapped runbook) are the deliverables the next four modules assume you have.

1. [Surface-level jailbreak eval with HarmBench shape](exercises/exercise-01-surface-level-jailbreak-eval-with-harmbench-shape.md) — stand up a HarmBench-shaped attack replay against the *application surface* with a payload-management contract that never checks harmful strings into the repo.
2. [Prompt-injection eval with AgentDojo](exercises/exercise-02-prompt-injection-eval-with-agentdojo.md) — run AgentDojo against your own agent (or a scaffolded copy of it), produce the `utility × security` scorecard, and derive at least one internal-suite task that mirrors a real product surface.
3. [Tool abuse and exfiltration eval](exercises/exercise-03-tool-abuse-and-exfiltration-eval.md) — build the sandboxed replay harness with canary tokens, out-of-scope-tool invocation detection, and a per-tool exfiltration-rate scorecard.
4. [Guardrail effectiveness with FP / FN report](exercises/exercise-04-guardrail-effectiveness-with-fp-fn-report.md) — score at least two guardrails (one OSS, one hosted) on a policy-aligned labelled set, publish the confusion matrix, the operating-point choice, and the cost per guarded request.
5. [OWASP LLM Top-10 mapping and runbook](exercises/exercise-05-owasp-llm-top10-mapping-and-runbook.md) — take the artefacts from exercises 01 – 04 and produce a single OWASP LLM Top-10-mapped safety-eval report plus a runbook that names each finding's owner, severity, and delegation target.

## Labs and quizzes

- Labs (see [`labs/`](labs)) build an end-to-end app-side safety-eval reference on top of the mod-102 instrumented backend and the mod-107 online loop: the jailbreak replay, the injection suite, the sandboxed tool-abuse harness, the guardrail scorecard, and the OWASP-mapped runbook. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary — the model-level / app-level split, the direct / indirect / mixed injection taxonomy, the ASR / ASR@k distinction, the utility × security scorecard shape, the guardrail confusion-matrix and cost-per-guarded-request, the refusal / over-refusal reconciliation, and the OWASP / ATLAS / SAIF mapping. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- **Harm model, red-team data, adversary modelling** are owned by the peer `ai-risk-engineer` (Governance family, level 30). This module's runbook names the finding types that get escalated there — real capability uplift, novel attack families, model-level dangerous behaviour → [ai-risk-engineer](../../../ai-risk-engineer-learning) (peer track, referenced when the org has one).
- **Dangerous-capability model-level evals** (weapons uplift, bio, cyber, autonomy) are owned by `model-evaluation-engineer`. This module measures the *application-surface* refusal against those categories; the underlying model-capability evaluation is a peer artefact.
- **Human review of the surviving alerts** consumes the safety-eval queue produced by chapter 07's runbook → [`mod-109-human-review-workflows`](../mod-109-human-review-workflows).
- **Eval-data platform** stores the jailbreak scorecard, injection scorecard, guardrail FP / FN report, and OWASP-mapped runbook as first-class tables → [`mod-110-eval-data-platform-slice`](../mod-110-eval-data-platform-slice).
- **Cost / latency / quality trade-off** reads the guardrail cost and latency line items from chapter 05 → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- **Program-owner posture** treats the OWASP-mapped runbook as one of the load-bearing artefacts of the release-gate-plus-online-monitor architecture → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
