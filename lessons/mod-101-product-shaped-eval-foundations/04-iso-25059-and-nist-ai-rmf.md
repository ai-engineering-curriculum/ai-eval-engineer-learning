# ISO/IEC 25059 and NIST AI RMF as Checklists

## Motivation

You will be asked, at least once, "did you consider fairness / robustness / user controllability / adaptability?" and the wrong answer to that question is "no." Two publicly available frameworks let you *systematically* check that the criteria list from chapter 02 covers everything the plan should cover:

- **ISO/IEC 25059:2023** — the AI-system extension of the SQuaRE quality model (ISO/IEC 25010). It defines the quality dimensions an AI system can be evaluated against. Good as a **per-surface completeness checklist**.
- **NIST AI Risk Management Framework (AI 100-1)** — a US NIST framework with four functions (Govern, Map, Measure, Manage) and dozens of categories/subcategories. Good as a **program-level scope frame** for where your eval work fits inside a broader risk-management posture. The **Generative AI Profile (NIST AI 600-1)** applies the same shape specifically to GenAI systems.

This module does not teach you to *own* a NIST or ISO conformance program (that shape belongs to the peer `ai-evaluation-engineer` in the Governance family and to `head-of-ai-governance`). It teaches you to *use* the two frames to make sure the eval plan you write does not have obvious gaps.

## ISO/IEC 25059 as a per-surface checklist

### What ISO/IEC 25059 is

[ISO/IEC 25059:2023 "Software engineering — Systems and software Quality Requirements and Evaluation (SQuaRE) — Quality model for AI systems"](https://www.iso.org/standard/80655.html) extends the ISO/IEC 25010 product-quality model with dimensions relevant to AI systems. The standard is paywalled; a short public overview is in the [ISO catalogue page](https://www.iso.org/standard/80655.html) and the freely available introductions of ISO/IEC 25010 for the SQuaRE base model.

<!-- needs-research: cross-check the current published dimension list of ISO/IEC 25059:2023 against the ISO catalogue in the next research cycle; if the standard has been amended, refresh this checklist accordingly. -->

At the level of *how you use it*, treat 25059 as an extension of the 25010 dimensions with additional AI-specific characteristics such as **functional adaptability**, **user controllability**, and **transparency** applied to AI-specific behaviour. The exact taxonomy is normative to the standard; consult the ISO document itself for authoritative definitions before you sign a plan that cites it.

### How to use it

For each surface, walk the 25010 quality characteristics as extended by 25059 and answer, per characteristic:

- **In scope for this plan?** (yes / no / deferred to peer)
- **Which criterion in chapter 02's decomposition covers it?**
- **What is the SLI?**
- **If deferred, to which peer role?**

A worked template (populate with your actual surface):

| ISO/IEC 25010/25059 characteristic | In scope? | Criterion (from ch. 02) | SLI | Owner / peer |
|---|---|---|---|---|
| Functional correctness | yes | Task-success rubric | Judge score ≥ 4 rate | this track |
| Functional appropriateness | yes | Intent-fit criterion | Rate of on-intent completions | this track |
| Functional adaptability (AI extension) | scope note | Handling of ambiguous inputs | Fallback / clarification rate | this track |
| Reliability | yes | End-to-end error rate | Per-surface error % | this track |
| Performance efficiency | yes | Latency, tokens per session | TTFT / p95, tokens / session | this track (`mod-111`) |
| Usability | yes | Regeneration & satisfaction | Regen rate, thumbs-down rate | product analytics + `mod-109` |
| Security | partial | Prompt-injection robustness | Injection-fixture pass rate | `mod-108`; depth to `ai-infra-security` |
| Compatibility | scope note | Streaming, browser support | Streaming-error rate | product infra |
| Maintainability | scope note | Fixture / harness health | Fixture flake rate | eval-platform (`mod-110`) |
| Portability | usually deferred | — | — | AI-platform team |
| Transparency (AI extension) | yes | Explanations / citations | Citation-precision score | this track (`mod-105` for RAG) |
| User controllability (AI extension) | yes | Regenerate / edit / opt-out | Availability & take-up | product |

Two rules of thumb when using 25059:

1. **Every row must be one of: `in scope`, `deferred (with named owner)`, or `not applicable (with one-sentence reason)`.** No blanks.
2. **The "not applicable" cases are the ones a reviewer will ask about.** Write the reason once, in the plan, so you do not re-litigate it.

## NIST AI RMF as a scope frame

### What the RMF is

The [NIST AI Risk Management Framework 1.0 (AI 100-1)](https://www.nist.gov/itl/ai-risk-management-framework) is a voluntary framework organised around four functions:

- **Govern** — set the risk-management posture (policies, roles, accountability).
- **Map** — understand the AI system, its context, and its potential impacts.
- **Measure** — assess the system against trustworthy-AI characteristics (validity/reliability, safety, security & resilience, accountability & transparency, explainability & interpretability, privacy-enhancing, fair with harmful bias managed).
- **Manage** — prioritise and respond to identified risks.

The **Generative AI Profile ([NIST AI 600-1](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile))** applies the framework specifically to generative AI, and lists risks (confabulation, dangerous / harmful content, misuse, IP, information integrity, information security, obscene content, value chain risks, etc.) with suggested actions per function.

### How this track uses it

This track — as a level-30 AI Engineering role — owns almost exclusively work inside **Measure** for AI *application* systems. That means:

- Your eval plan is a **Measure-function artefact.**
- The **Govern** function is owned by the Governance-family peer track (`ai-evaluation-engineer` at level 25, `head-of-ai-governance` above).
- **Map** is a joint activity with the product team and the risk peer; your eval plan should be able to point at the map artefact but does not author it.
- **Manage** consumes your eval evidence to decide whether to accept, mitigate, or reject risks. You provide evidence; you do not make the risk-acceptance decision.

Use the RMF's Measure categories as a **program-level scope frame**:

| RMF Measure category (paraphrased) | Where in your plan |
|---|---|
| Appropriate metrics identified and applied | Chapter 02 criteria table |
| Trustworthy characteristics measured | The five families, cross-checked with 25059 |
| Measurements over time (drift, regression) | Chapter 03 online loop; `mod-107` |
| Feedback from context / users incorporated | `mod-109` human review; product feedback loops |
| Risks identified in Map are measured in Measure | Cross-reference to the Map artefact |

If your plan cannot answer "which RMF Measure category does this SLI serve," the SLI is either mis-scoped or belongs in a governance artefact you should not be authoring.

### The Generative AI Profile in one paragraph

The [Generative AI Profile (AI 600-1)](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile) enumerates suggested actions per RMF function specifically for GenAI risks. Two things are practically useful for a plan:

- The risk list is a **completeness prompt** — walk it and ask "does my surface plausibly expose this risk? if yes, do I have a criterion?"
- The suggested-actions list often names an activity — e.g., testing for prompt injection, monitoring for confabulation — and your plan can point directly at the module in this track that covers it (`mod-108` for injection, `mod-105` for confabulation-as-groundedness, `mod-107` for monitoring).

## When to *not* rely on these checklists

- **They do not tell you what the SLO should be.** They tell you which dimensions to consider. You still have to write the numeric objective with the product team.
- **They are not a substitute for the peer Governance track's shape.** A NIST-aligned or ISO-aligned *release-assurance* posture is Governance-family work. Your plan produces the *technical* evidence.
- **They can become checklist theatre.** If every row is "yes, in scope" with vague criteria, the plan is worse than one that admits scope decisions honestly.

## Example: applying the two frames to the meeting summariser

- **ISO/IEC 25059 pass:** walking the extended 25010 dimensions, the plan gains a *user controllability* criterion ("user can regenerate and edit; edit uptake is measured") and a *transparency* criterion ("action items link to the transcript span") that the raw spec was silent on. Both get added to the criteria table with SLIs.
- **NIST AI RMF Measure pass:** the plan gains a **drift-watch SLI** on faithfulness (Measure "measurements over time"), and an explicit note that HR-content leak measurement here does *not* discharge Govern-function obligations — those belong to the Governance peer track.
- **AI 600-1 pass:** the risk *confabulation* maps to the faithfulness SLI; *information integrity* maps to the attribution criterion; *information security* points at the injection-fixture set (`mod-108`); *IP* and *obscene content* are marked **not applicable, one-sentence rationale, PM-acknowledged.**

## Summary

- **ISO/IEC 25059** is a per-surface **completeness checklist**. Walk its dimensions; every row is `in scope`, `deferred to named peer`, or `not applicable with one-sentence reason`.
- **NIST AI RMF** is a **program-level scope frame**. Your plan is a **Measure**-function artefact; **Govern** and **Manage** belong to the Governance peer track.
- The **Generative AI Profile (AI 600-1)** is a completeness prompt for generative-AI-specific risks.
- Neither framework picks your SLOs for you. They keep you honest about coverage.

The next chapter looks at the opposite failure mode: what happens when a proposed evaluation *is* famous, well-cited, and completely wrong for your surface.
