# Why App-Side Safety Eval, and Where It Hands Off

## Motivation

There is a version of "safety eval" that lives at the model layer: measure whether a base model, given a raw prompt through an unwrapped API, produces content that violates the provider's usage policy. That work is real and important — but it is not this track's job. It is owned by `model-evaluation-engineer` on the model side and by `ai-risk-engineer` on the harm-model side. Both peers are level-30 tracks with their own curricula.

The work this module owns is the version of "safety eval" that only exists *when a user is on your product*. Concretely:

- A jailbreak that succeeds against the raw `gpt-4o` API may be filtered by your app's system prompt, your input classifier, your retrieval filter, or your output moderator. **The number that matters is the attack success rate after all of those layers, on your surface, on your endpoint.** The raw-model number is peer-owned.
- A prompt injection embedded in a retrieved document is only exploitable if your agent trusts the retrieved document, wires it into the same context as the user turn, and has tools that can act on the injected instruction. **The number that matters is the exploitation success rate against your specific agent configuration.** The base model's susceptibility, in isolation, is peer-owned.
- A guardrail catches or misses based on the operating point you chose and the policy you wrote for your product surface. **The number that matters is the false-positive rate (over-refusal that hurts users) and the false-negative rate (safety escapes that hurt users) at that operating point, on your traffic.** The guardrail's raw ROC is a vendor artefact.
- A refusal is either well-calibrated to your written safety policy or it is not. **The number that matters is the row-by-row reconciliation between the model's refusal decisions and the policy the product owner signed.** The policy itself is a product artefact; the reconciliation is the eval-team artefact.
- Findings are only actionable when the on-call and the product owner can map them to a framework the org already recognises. **The number that matters is the OWASP / ATLAS / SAIF-tagged runbook entry with a named owner.** A finding with no framework tag and no owner is a finding that quietly ages out.

This module walks the mechanism for those five measurements. It reuses the online-eval loop from mod-107 (chapter 07 will show the wiring), the trace instrumentation from mod-102 (spans need safety-attribute enrichment), and the LLM-as-judge tier router from mod-104 (safety rubrics are among the ones scored). What is new is the *attack side* and the *policy side*: how to score against attackers you do not want to check into your repo, and how to reconcile refusals against a written safety policy.

This chapter fixes the vocabulary the rest of the module reuses — **app-surface vs model-surface**, **jailbreak**, **prompt injection** (direct / indirect / mixed), **guardrail**, **refusal / over-refusal**, **red-team data**, **payload-management contract** — and states the delegation triangle the module operates inside.

## Core concepts

### The app-surface / model-surface split, and who owns what

The single most common failure mode of an app-safety program is confusing "the base model refused" with "the app refused." Different owners, different mitigations, different runbooks.

| Surface | Owner | What "an eval" measures | Example |
|---|---|---|---|
| **Model surface** — the raw provider API with an empty system prompt | `model-evaluation-engineer` (peer track) | Attack success on the base model; dangerous-capability elicitation; refusal calibration against the provider's usage policy | HarmBench run against unwrapped `claude-3-5-sonnet` |
| **App surface** — your endpoint, with the full system prompt, retrieval, tools, and guardrails in place | This module | Attack success *after all your defences*; injection exploitation on *your* agent; guardrail FP / FN at *your* operating point; refusal calibration against *your* safety policy | HarmBench-shaped replay against `POST /v1/support-bot/chat` |
| **Harm-model / red-team data production** — deciding which harms matter, generating adversary personas and payloads | `ai-risk-engineer` (peer track) | Threat modelling artefacts; adversary personas; hazard categorisation | "Which of the OWASP LLM Top-10 apply, at what severity, for our surface?" |

The three surfaces share vocabulary but not artefacts. An app-eval report cites the risk-team's threat model and the model-eval-team's baseline; it does not re-derive them. When your `POST /v1/support-bot/chat` fails an app-surface eval, the on-call knows to look for a defence-in-depth breakdown *in your stack*; when the model-surface eval regresses, the on-call knows to look at the provider snapshot.

Get this wrong and you either duplicate peer work (expensive, and the peer will not accept your version as authoritative anyway) or leave a gap where nobody owns the number (worse — the surface fails, the incident retro asks whose eval should have caught it, and everyone points at each other).

### The five axes this module measures

Every subsequent chapter builds one axis. Together they cover the app-surface safety-eval surface.

| Axis | Chapter | What it scores | Typical suites / tools |
|---|---|---|---|
| **Jailbreak resistance** | 02 | Attack success rate on refusal-eligible prompts hitting your surface | HarmBench-shaped replays; PAIR / GCG-style automated attacks; garak; PyRIT |
| **Prompt-injection robustness** | 03 | Exploitation rate for direct, indirect, and mixed injections in RAG / tools / memory | AgentDojo; InjecAgent; derived internal suites |
| **Tool-abuse containment** | 04 | Rate of out-of-scope tool invocation and data-exfiltration success in a sandbox | AgentDojo tool-abuse tasks; internal sandbox with canary tokens |
| **Guardrail effectiveness** | 05 | FP / FN of each guardrail at its operating point plus cost and latency | Llama Guard, NeMo Guardrails, Guardrails AI, provider moderation APIs |
| **Refusal / over-refusal calibration** | 06 | Refusal decisions vs the written safety policy, row by row | XSTest, OR-Bench-shaped suites, internal policy-alignment sets |

Chapter 07 stitches all five into an OWASP LLM Top-10-mapped runbook that a product owner and a security reviewer can each read.

### The delegation triangle the eval team lives inside

```
                    ┌──────────────────────────┐
                    │   ai-risk-engineer       │
                    │   (Governance family)    │
                    │                          │
                    │ owns:                    │
                    │ - harm model             │
                    │ - adversary personas     │
                    │ - red-team data          │
                    │ - hazard categorisation  │
                    └──────────────────────────┘
                              ▲    │
                    threat    │    │  hazard
                    model     │    │  categories,
                    citations │    │  adversary
                              │    │  personas
                              │    ▼
                    ┌─────────────────────────┐
                    │  ai-eval-engineer       │
                    │  (this module)          │
                    │                         │
                    │ owns:                   │
                    │ - app-surface replay    │
                    │ - injection suite       │
                    │ - sandbox harness       │
                    │ - guardrail scorecard   │
                    │ - refusal calibration   │
                    │ - OWASP-mapped runbook  │
                    └─────────────────────────┘
                              ▲    │
                    model-    │    │  app-surface
                    surface   │    │  failure rate,
                    baseline  │    │  requests for
                              │    │  model-level
                              │    │  diagnosis
                              │    ▼
                    ┌──────────────────────────┐
                    │ model-evaluation-        │
                    │ engineer                 │
                    │ (Model-Development       │
                    │  family)                 │
                    │                          │
                    │ owns:                    │
                    │ - dangerous-capability   │
                    │   evals at model level   │
                    │ - refusal calibration on │
                    │   the raw model          │
                    │ - training-time safety   │
                    │   experiments            │
                    └──────────────────────────┘
```

Two arrows are contract lines the eval team maintains:

- **In from the risk team.** The harm model and adversary personas are *inputs*. This module's suites cite them (chapter 03's injection categories reference the risk-team's exfiltration threat; chapter 04's out-of-scope tool set is derived from the risk-team's adversary persona list). The eval team does not invent the threat model.
- **Out to the model-eval team.** When the app-surface finding survives the eval team's defence-in-depth investigation and looks like model-level dangerous-capability behaviour (chapter 07's runbook criteria), the finding is *escalated*, not fixed in the app. The model-eval team then runs the model-level assessment. This is the same escalation shape mod-107 defined for drift alerts that the on-call cannot resolve.

Get the arrows wrong and the org either accumulates duplicate work (this track and the model-eval track scoring the same base-model behaviour with slightly different suites) or accumulates gaps (nobody owning the "the app failed even though the model would not have").

### Vocabulary the module reuses

Six terms recur across the chapters. Fix them once here; use them consistently in your repo.

- **Jailbreak.** Any prompt or prompt sequence that induces the model to violate its refusal policy on a category the policy prohibits. Includes handcrafted prompts (DAN-style role-play, translation attacks, encoding attacks), automated-search attacks (GCG's adversarial suffix, PAIR's iterative rewriter), and multi-turn attacks. Chapter 02's HarmBench-shaped replay scores this on your app surface.
- **Prompt injection.** Any input that causes the agent to follow instructions from a source other than the trusted principal (the user or the developer). **Direct** = adversarial content in the user turn itself. **Indirect** = adversarial content in a document the retriever fetched or a tool the agent called; the user did not write the payload. **Mixed** = the user turn is benign, the retrieved document nudges the model to fetch a poisoned URL whose content then instructs. Chapter 03's AgentDojo / InjecAgent scoring walks this taxonomy.
- **Tool abuse.** The agent invokes a tool it should not have, or supplies arguments that leak or exfiltrate data. Chapter 04's sandbox measures this by wiring canary tokens and monitoring egress. Distinct from injection — a tool call can be *induced by* injection or it can be a native failure ("the agent decided on its own to send this data to search").
- **Guardrail.** Any layer that observes the input or output and can block, rewrite, or annotate. Includes rule-based filters (regex, keyword), classifier-based filters (Llama Guard, Presidio, Perspective API), programmable frameworks (NeMo Guardrails, Guardrails AI), and provider moderation APIs (OpenAI Moderation, Azure AI Content Safety, Google Model Armor). Chapter 05's scorecard measures each guardrail's FP / FN plus its cost and latency.
- **Refusal / over-refusal.** A **refusal** is the model declining to complete a request. **Over-refusal** is refusing a request that the written safety policy permits (a benign nurse asking about medication interactions, a fiction writer asking about weapons in the abstract). Chapter 06's reconciliation measures both.
- **Payload-management contract.** The rule that harmful strings never enter the repo. Attack suites are consumed by *reference* — a HarmBench behaviour id, a hash of a canonical prompt, an offline-only sealed archive — never by having the raw payload in a `.jsonl` file that `git grep` will surface. Chapter 02 formalises this; every subsequent chapter respects it.

Two more terms carry over from mod-107 and mod-104 without being redefined here: **online-eval loop** (mod-107 chapter 01) and **judge-tier router** (mod-104 chapter 05). This module reuses both; nothing about the *mechanism* is new. What is new is the *content* — safety rubrics, adversarial inputs, guardrail scorers.

### Why the app-surface number is the number that matters

Three grounded reasons the model-surface number, alone, is not enough.

- **Layered defences make the numbers materially different.** A base model with a documented 15 % attack success rate on a jailbreak suite, wrapped in a system prompt that describes the product's boundaries, plus an input classifier at the front, plus an output guardrail at the back, plus a retrieval filter for the RAG cases — that combined stack routinely produces app-surface attack success rates one or two orders of magnitude lower. Reporting the raw number to the product owner as "our surface" is wrong; ignoring the raw number is also wrong (the layers are not static — someone will delete the input classifier for latency; the app-surface number will jump; the model-surface number will not).
- **Injections happen only in a shape the model surface cannot even test.** A raw-model injection eval sends the injection in the user turn. A RAG-shaped injection eval sends the injection in the retrieved document that arrives *inside* a `<context>` tag alongside the user turn. Different models handle the two differently. The model-surface number is the wrong prior for the app-surface number.
- **Guardrails and policies are yours, not the model's.** A guardrail decision is a function of the operating point you chose, the categories your policy names, and the classifier your team wired in. The model-eval team cannot report this number for you; the number does not exist at the model surface.

None of this contradicts the model-surface eval — both matter. It does mean the two live in different reports with different owners.

### The eval loop, unchanged

The mechanism this module runs on is the mod-107 online-eval loop with three adjustments:

- **Stricter thresholds.** A drift-monitor alert on faithfulness at a soft-fail threshold is a signal for the on-call to investigate at their pace. A drift-monitor alert on refusal rate for the "self-harm" category is a page-immediately signal; the runbook is different; the threshold is often zero-tolerance. Chapter 06 formalises which thresholds are soft-fail vs hard-fail.
- **Zero-tolerance safety floors on the canary gate.** Mod-107 chapter 04 defined the cohort gate with per-cohort deltas; the safety carve-out was named but not walked. This module walks it: certain metrics (unsafe-content leakage rate, tool-abuse rate on sensitive tools) have an absolute floor, not a delta, and no soft-fail. A single unsafe response in the canary is enough to halt the rollout.
- **Attack suites replace the fixture replay for a fraction of the eval budget.** The mod-106 fixture replay tests representative user traces. The safety half of the same gate needs *adversarial* traces — the fixture pipeline does not produce these. Chapter 02 walks the payload-management contract that lets you consume these suites without the payloads landing in the repo.

The rest is the same loop. The scored-row schema from mod-107 chapter 02 gains a few columns (`attack_family`, `guardrail_decision`, `refusal_category`, `policy_reconciliation_verdict`); the aggregator, drift monitor, canary gate, and dashboards all read the same rows.

### What this module is not

- **The harm model.** Which harms matter, for whom, at what severity, is owned by `ai-risk-engineer`. This module's suites read from that document; they do not produce it.
- **Adversary personas or red-team data generation.** The threat actors, their motivations, their capabilities are also risk-team artefacts. This module runs the replays; it does not author the personas.
- **Dangerous-capability model-level evaluations.** Weapons uplift, biosecurity, cybersecurity, autonomy — the base-model capability numbers are `model-evaluation-engineer` artefacts. This module measures whether the *app* surfaces those capabilities; the underlying model number is peer-owned.
- **The provider's usage policy.** OpenAI, Anthropic, Google, Meta, Mistral each publish their own. Your policy sits *alongside* those (usually a subset). Chapter 06 walks the reconciliation; the policy itself is a product / legal artefact.
- **Human review of surviving alerts.** When an alert cannot be resolved by the runbook, it lands in the mod-109 queue. This module ends at "escalate with framework tag and severity"; mod-109 defines the queue.
- **The eval data platform.** The scored rows, the attack-suite versioning, the guardrail-run history are all mod-110. This module treats them as substrate.

### The end-of-module posture

By the end of this module you own the app-side safety half of the eval program. Concretely, you can:

- Ship a HarmBench-shaped attack replay against your surface without harmful payloads ever entering the repo.
- Ship an AgentDojo / InjecAgent-shaped injection eval that scores both `utility` (the agent still does its job on benign traffic) and `security` (it does not act on injected instructions) as a joint metric.
- Ship a sandboxed tool-abuse harness that catches out-of-scope tool invocations and data-exfiltration attempts using canary tokens.
- Ship a guardrail scorecard for at least two guardrails with FP / FN, cost per guarded request, and latency at the operating point.
- Ship a refusal / over-refusal reconciliation against a written safety policy with per-row verdicts.
- Ship an OWASP LLM Top-10-mapped runbook with named owners, severities, and delegation targets.

The chapters build these one by one. Chapter 02 starts with the jailbreak axis and the payload-management contract that everything else in the module respects.

## Summary

- The **app-surface / model-surface split** is the load-bearing distinction of this module. The app number is what matters for the product; the model number is peer-owned.
- Three peers form a **delegation triangle**: `ai-risk-engineer` (harm model, red-team data), `ai-eval-engineer` (this module — app-surface measurement), `model-evaluation-engineer` (model-level dangerous-capability evals). Contracts, not overlap.
- The module covers **five axes**: jailbreak resistance (ch. 02), prompt-injection robustness (ch. 03), tool-abuse containment (ch. 04), guardrail effectiveness (ch. 05), refusal / over-refusal calibration (ch. 06). Chapter 07 maps findings to OWASP LLM Top-10, MITRE ATLAS, and SAIF.
- Six vocabulary terms recur: **jailbreak**, **prompt injection** (direct / indirect / mixed), **tool abuse**, **guardrail**, **refusal / over-refusal**, **payload-management contract**.
- The **online-eval loop** from mod-107 is the mechanism, with stricter thresholds, zero-tolerance safety floors on the canary gate, and attack suites replacing part of the fixture replay.
- The module *does not* produce the harm model, the adversary personas, the dangerous-capability model-level evals, or the provider usage policy — each is peer-owned.

Chapter 02 walks the jailbreak axis and the payload-management contract that every subsequent chapter respects.
