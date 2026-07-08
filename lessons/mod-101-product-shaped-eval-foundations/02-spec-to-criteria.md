# From Product Spec to Eval Criteria

## Motivation

Product specs are written for humans building the thing; they are not written for people evaluating it. A spec will tell you the *what* ("draft a follow-up email from the last call") and part of the *why* ("cut sales-rep prep from 20 minutes to 2"), but rarely the *how do we know we did it right*. Turning that spec into criteria that a regression suite can compare against and a sampled-trace loop can watch is the first hard step of every eval program.

This chapter walks through the pattern this track uses to do that decomposition: **claims → criteria → SLIs → SLOs**, ending with a short worked example. It borrows the [Google SRE workbook](https://sre.google/workbook/implementing-slos/) vocabulary from chapter 01 and grounds product-quality thinking in the ISO/IEC 25059 dimensions covered in chapter 04.

## Core concepts

### Read the spec for four things

When you first read a product spec, tag it for four categories of information:

1. **User intent claims** — "the user wants to X, we will do Y for them." These become **task-success** criteria.
2. **Correctness / grounding claims** — "the output is based on the meeting transcript / the retrieved doc / the ticket history / the CRM record." These become **groundedness** criteria and are RAG-triad candidates ([RAGAS paper, Es et al. 2023](https://arxiv.org/abs/2309.15217)).
3. **Constraint claims** — "we will not do Z; we will always refuse W; we will not exceed N tokens; we will respond within M ms; we will cost less than $C per invocation." These become **safety, cost, and latency** criteria.
4. **Distribution claims** — "our users are enterprise support agents on desktop; ~70% of traffic is English." These are not criteria themselves, but they define the sampling frame your offline suite and your online sampled-trace loop must match.

If any of these four is missing, that is a real finding you file with the PM before you write any eval, not a defect you paper over with a plausible default.

### From claim to criterion

Each tagged claim decomposes as:

```
CLAIM  ("summaries are based on the meeting transcript")
  ↓
CRITERION  ("no unattributed statement outside the retrieved transcript spans")
  ↓
SLI  ("share of judged summaries with faithfulness score ≥ 4 on a 1–5 rubric")
  ↓
SLO  ("≥ 92% weekly, on the sampled-trace stream")
  ↓
OFFLINE FIXTURE and/or ONLINE METRIC  (see chapter 03)
```

Two rules that will save you rework:

- **Every criterion must be *falsifiable* by a specific test procedure.** "Summaries are high quality" is a claim; "no invented meeting attendee" is a criterion. If you cannot describe the procedure that would score it, it is not yet a criterion.
- **Every criterion must have a numeric SLI.** Even human-review criteria produce a number (share of samples judged compliant). If you cannot express it numerically, you cannot regression-test it and you cannot draw a rollout gate against it.

### The five families, per surface

For each surface, expect to end up with at least one criterion in each of the five families from chapter 01. If the spec doesn't imply one, ask.

| Family | Typical criteria at product altitude | Typical SLI shape |
|---|---|---|
| Task success | Correct final answer; correct final tool call; user goal satisfied within N turns | Rate / share, sometimes per-cohort |
| Groundedness | Faithfulness to retrieved / user-provided source; answer relevancy; context precision/recall (RAG-triad) | Judge score, share ≥ threshold |
| Safety | Refuses in-scope harmful requests; does not over-refuse benign; robust to prompt injection on the surface; safe tool use | Rate, per-attack-family breakdown |
| Cost | Tokens per session; judge-cost per evaluated trace; downstream API cost | $ / trace, tokens / trace |
| Latency | TTFT, TPOT (streaming), end-to-end p50 / p95 | ms, per quantile |

`mod-105` (RAG eval), `mod-108` (safety and guardrails), and `mod-111` (cost/latency/quality trade-offs) each deepen their family. This module builds only the frame.

### The "so what" test

Before you commit any criterion to a plan, ask two questions:

1. **Actionability** — "if this SLI drifts, what does the team do?" If the answer is "nothing, we would just look at it," it does not belong in the plan.
2. **Attribution** — "if this SLI moves, can we tell which change caused it?" If the criterion mixes retrieval quality, generation quality, and safety guardrails into a single number, you have built a debugging headache. Split it.

The RAG-triad exists precisely because a monolithic "quality" score cannot separate a retrieval regression from a generation regression (see chapter on RAG eval in `mod-105`).

### Naming the SLOs

Two conventions save you political capital later:

- **Name the window and the population.** "≥ 92% weekly on the sampled-trace stream, English traffic" is legible. "≥ 92%" is not.
- **Split ship-gate from watch-only.** Not every SLI needs to gate a release. Some are informational (drift watch). Say which is which up front so no one gets surprised by a red build later.

## Example: a "meeting summariser" surface

Consider a short spec excerpt:

> *"Meeting Summariser" turns a Zoom transcript into a 200-word summary and a bulleted action-item list. It appears as a card in the meeting recap email 5 minutes after the meeting ends. It must attribute action items to the meeting attendee who accepted them. It must not surface sensitive HR discussion. Users can regenerate up to twice.*

Tagging the spec:

| # | Excerpt | Type | Family |
|---|---|---|---|
| 1 | "turns a Zoom transcript into a 200-word summary" | Intent | Task success |
| 2 | "and a bulleted action-item list" | Intent | Task success |
| 3 | "It must attribute action items to the meeting attendee who accepted them." | Grounding | Groundedness |
| 4 | "It must not surface sensitive HR discussion." | Constraint (refusal) | Safety |
| 5 | "5 minutes after the meeting ends" | Constraint (latency) | Latency |
| 6 | "Users can regenerate up to twice." | Cost signal | Cost |

Decomposing #3 all the way to an SLO:

```
CLAIM       "attribute action items to the attendee who accepted them"
CRITERION   "every action item names an attendee who appears in the transcript"
            "the attributed attendee is the one who verbally accepted the item"
SLI         share of action items scored ≥ 4 on a 1-5 attribution rubric by
            a judge model, calibrated against a 100-item human gold set
SLO         ≥ 95% weekly on sampled traffic; ≥ 98% on the offline regression
            fixture pre-merge
GATE        offline SLO is a merge gate; online SLO is a watch-and-page target
DEFERRED    full judge-vs-human calibration methodology → model-eval owner
```

Decomposing #4 (safety) similarly:

```
CLAIM       "must not surface sensitive HR discussion"
CRITERION   "no summary or action item reproduces content from an HR-tagged
            segment of the transcript"
SLI         share of summaries flagged as leaking HR content by a rubric-based
            judge on an HR-seeded fixture; false-positive rate on benign
            traffic
SLO         0 leaks on the HR-seeded fixture pre-merge (hard gate);
            ≤ 0.5% false-positive rate on benign traffic
GATE        both are merge gates
DEFERRED    prompt-injection stress against this refusal → mod-108
            harm-model for what counts as "sensitive HR" → ai-risk-engineer
```

Notice the plan already declares two peer-owner delegations. This is normal, and doing it early prevents scope creep.

## Applying the ISO/IEC 25059 checklist as a completeness pass

Once you have the criteria table, run it against the ISO/IEC 25059 quality dimensions (chapter 04) as a "did we miss a family?" pass. If your plan has nothing on functional adaptability, robustness, or user controllability, that is either a defensible scope decision (write down *why*) or a gap.

## Summary

- Read the spec for four things: intent claims, grounding claims, constraint claims, distribution claims.
- Decompose each claim to a **criterion → SLI → SLO** ladder with a named window, population, and gate type.
- Cover all five criteria families per surface; if the spec is silent on one, flag it back to PM.
- Every criterion must be **falsifiable** by a procedure and **numeric** in its SLI.
- Apply the "so what" test: is the criterion **actionable** and **attributable**?
- Name the peer owners for depth you are not measuring (`model-evaluation-engineer`, `ai-risk-engineer`, `ai-evaluation-engineer`).

The next chapter decides *where* each of these criteria is measured: offline, online, or handed off.
