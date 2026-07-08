# What "Product-Shaped" Evaluation Means

## Motivation

If you have already done model evaluation, application evaluation looks disorienting. The unit under test isn't a model checkpoint — it's a *surface*: a specific user-visible interaction (a chat, a completion, a tool call, a retrieval-augmented answer) with a prompt, a chain, a retriever, a set of tools, decoding settings, guardrails, and a business context wrapped around a model call. Two surfaces backed by the same model can have completely different quality profiles, completely different failure modes, and completely different SLOs. Aggregate leaderboard numbers do not tell you which of those two surfaces is safe to ship.

This first chapter fixes the vocabulary the rest of the module relies on: *surface*, *criteria*, *SLO/SLI*, *altitude*, and *ownership*. It also draws the line between what this track evaluates and what the peer `model-evaluation-engineer` and `ai-evaluation-engineer` (Governance family) tracks own — because the answer to "who measures that?" is a first-class part of every eval plan you will write.

## Core concepts

### Surface

A **surface** is the smallest unit of an AI product you can meaningfully evaluate as a whole. It has:

- A **user intent contract** — what the user is trying to accomplish when they land here.
- A **runtime shape** — single-turn completion, multi-turn chat, retrieval-augmented answer, tool-calling agent, streamed suggestion, etc.
- A **model + prompt + chain configuration** — including retriever, tools, guardrails, and decoding.
- A **product SLO** — the quality/cost/latency/safety promise the surface makes.
- An **owner** — usually a product team plus an ML/AI engineer.

Everything downstream in this module is scoped **per surface**. "Our chatbot" is not a surface; "the ‘summarise this thread' action inside the inbox" is. Two surfaces sharing a model but not sharing a prompt, retriever, or tool set need two eval plans.

### Altitude

We use **altitude** to describe where an evaluation lives in the stack:

| Altitude | Question it answers | Owner |
|---|---|---|
| Model altitude | "Which checkpoint is better at this task family?" | `model-evaluation-engineer` |
| Application altitude | "Does this surface, as configured, meet its SLO for real users?" | `ai-eval-engineer` (this track) |
| Governance altitude | "Can we ship this under the applicable framework?" | `ai-evaluation-engineer` (Governance family) |

Application altitude is where a bad retriever, an over-long system prompt, a tool schema mismatch, a stray guardrail false-positive, or a decoding-setting drift will actually surface. Model altitude will not see any of that. Governance altitude cares about it but reasons about it as evidence, not as a knob to turn.

### Five criteria families

Every product-shaped eval plan you write in this track will touch at least these five criteria families:

1. **Task success** — did the surface do the thing the user hired it to do?
2. **Groundedness / faithfulness** — did the surface stay tethered to the source material or to reality (especially in RAG and tool-use)?
3. **Safety** — did it refuse what it should refuse, and not over-refuse; is it robust to prompt injection and to tool abuse?
4. **Cost** — token spend, tool spend, judge spend per unit of user value.
5. **Latency** — TTFT, TPOT (for streaming), and end-to-end p50/p95 for the interaction users actually see.

Chapter 02 unpacks how to derive concrete criteria in each family from a product spec. Chapters 05 and 06 dig into RAG-specific groundedness (RAG-triad) and safety measurement, respectively, in the modules that own those depths (`mod-105`, `mod-108`).

### SLO / SLI / SLA

The vocabulary comes from the [Google SRE workbook, chapter 2](https://sre.google/workbook/implementing-slos/). We reuse it verbatim:

- **SLI** — a *service level indicator*: a directly measurable quantity (e.g., "share of ‘summarise this thread' answers that a judge scores ≥4 on faithfulness").
- **SLO** — a *service level objective*: the target on an SLI (e.g., "≥ 92% of summaries score ≥4 on faithfulness, measured on a sliding 7-day window of sampled production traffic").
- **SLA** — a *service level agreement*: a contract externalising an SLO to a customer with a remedy attached.

An eval plan authored in this track defines SLIs and proposes SLOs. SLAs are a product/legal decision that consumes those objectives.

### What this track *does not* own

Because the vocabulary overlaps, it's worth writing the exclusions down early:

- **Model-checkpoint comparison methodology**, benchmark construction, statistical estimation depth, judge-vs-human calibration methodology (Cohen's κ, Spearman, bootstrap CIs at methodology depth), cross-modality model-eval architecture, and MLPerf-style serving benchmarks are owned by the peer `model-evaluation-engineer` (level 30, ML Engineering family). This module *consumes* those and cites the interface.
- **Audit trails**, third-party evaluator interfaces, and regulator-facing evidence packaging are owned by the peer `ai-evaluation-engineer` (level 25, Governance family). This module produces the technical evidence they package.
- **Alignment-risk methodology and red-team dataset generation** are owned by the peer `ai-risk-engineer` (level 30). `mod-108` in this track measures on the *product surface*.

Every eval plan you write should end with a **"deferred to"** section that names those peer owners for the items you did not measure yourself. Chapter 06 shows the template.

## Example: the same model, three surfaces

Suppose your product uses a single frontier chat model for three separate features:

| Surface | Runtime shape | Primary criteria worth watching | Cost / latency ceiling |
|---|---|---|---|
| Inbox: "summarise this thread" | Single-shot with retrieved history | Faithfulness, task success, PII refusal | Low latency, cheap enough to run on every open |
| Sales: "draft a follow-up email from this call" | Multi-turn with tool call to CRM | Task success, groundedness against the transcript, tone-fit, tool-call correctness | Medium latency, medium cost |
| Support: "answer with docs and open a ticket" | Retrieval + tool-calling agent | RAG-triad, tool-call correctness, refusal on out-of-scope, refund policy safety | Bounded step count, tight cost per session |

A model-altitude report ("checkpoint B is 2 points better on MT-Bench") tells you nothing useful about any of these three surfaces. Each surface deserves its own eval plan, its own offline regression suite, its own online sampled-trace eval, and its own SLO ledger. This is what the rest of the module teaches you to write.

## Summary

- **Surface** is the unit of evaluation at application altitude.
- Every surface has an intent contract, a runtime shape, a config, an SLO, and an owner.
- Five criteria families cover almost every product-shaped eval plan: **task success, groundedness, safety, cost, latency**.
- Reuse Google SRE vocabulary — **SLI/SLO/SLA** — verbatim; do not reinvent it.
- Model altitude, application altitude, and governance altitude are owned by different roles. Every eval plan you write names the peer owner for items you are not measuring.

The next chapter turns a product spec into concrete criteria in these five families.
