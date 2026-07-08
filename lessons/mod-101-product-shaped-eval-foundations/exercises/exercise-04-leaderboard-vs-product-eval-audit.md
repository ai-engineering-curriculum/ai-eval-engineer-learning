# exercise-04: Leaderboard vs Product-Eval Audit

**Estimated effort:** 2 hours

## Objective

Practice the skill of turning down a proposed benchmark on defensible, PM-legible grounds — and of accepting or adapting one when the mismatch is not fundamental. You will run the eight-question rejection template from chapter 05 against three candidate public benchmarks, classify each as `accept / adapt / reject` for a chosen surface, and write the memo you would send back to the stakeholder proposing them.

## Prerequisites

- Exercise-01 completed for at least one surface (so you have a criteria table).
- Chapter 05 of this module.
- Time to read at least the abstract and results section of each candidate benchmark's paper or README.

## Choose the surface and the candidate benchmarks

**Surface:** use the same one as exercise-01, or one of the three from that exercise (sales follow-up email, PR summariser, support-triage RAG+tool agent).

**Candidate benchmarks:** pick exactly three from this list, chosen to *stress* the accept / adapt / reject decision, not to make it easy. Deliberately pick one obvious mismatch, one plausible adaptation, and one that might be an accept:

- [MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685) (freeform chat, LLM-as-judge).
- [MMLU](https://arxiv.org/abs/2009.03300) (multiple-choice academic knowledge).
- [GSM8K](https://arxiv.org/abs/2110.14168) (grade-school math word problems).
- [SWE-bench](https://arxiv.org/abs/2310.06770) and [SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) (software engineering task benchmark).
- [τ-bench](https://arxiv.org/abs/2406.12045) (tool-use / customer-service-shape agent benchmark).
- [WebArena](https://arxiv.org/abs/2307.13854) (browsing agent benchmark).
- [GAIA](https://arxiv.org/abs/2311.12983) (general assistant benchmark).
- [BFCL — Berkeley Function-Calling Leaderboard](https://gorilla.cs.berkeley.edu/leaderboard.html) (function-calling correctness).
- [ToolBench](https://arxiv.org/abs/2307.16789) (tool-use agent benchmark).
- [RAGAS](https://arxiv.org/abs/2309.15217) suite for RAG (application-shaped evaluation library / benchmark templates).
- [AgentBench](https://arxiv.org/abs/2308.03688) (multi-environment agent benchmark).

If your workplace's benchmark portfolio has real proposals under discussion, use those instead of this list, redacting anything sensitive before you commit.

## Requirements

Produce a single markdown file `benchmark-audit-<surface-slug>.md` containing four sections.

### 1. Surface recap (5 lines)

Restate the surface, its user intent, its runtime shape, and one sentence naming what a "good answer" looks like on your production distribution. This is what the audit is anchored against.

### 2. Eight-question audit table

For each of your three chosen benchmarks, fill the audit table:

| # | Question | Answer for benchmark |
|---|---|---|
| 1 | Distribution — how similar are the benchmark's inputs to a representative sample of my surface's production traffic? | ... |
| 2 | Contamination — is the test set public, and what evidence is there that the candidate models did / did not train on it? | ... |
| 3 | Scaffold — bare model, or a scaffold resembling my surface's? | ... |
| 4 | Metric — does the metric penalise the failure my users will notice? | ... |
| 5 | Aggregation — is per-slice / per-tail reporting available? | ... |
| 6 | Prompt template — published and matching my surface's template? | ... |
| 7 | Judge — if LLM-as-judge, which judge, with which bias controls? | ... |
| 8 | Recency — release date; has the frontier caught up? | ... |
| — | Verdict | `accept` / `adapt` / `reject` |

Each cell should be a full sentence, not a yes/no. Where you cannot answer without more research, say so — an honest "unknown, would need to check X" is a legitimate answer.

### 3. Stakeholder memo

Write a ≤ 300-word memo you would send to the stakeholder who proposed all three benchmarks. Structure:

- One-sentence recap of what they proposed.
- The verdicts (accept / adapt / reject) as three bullets, one sentence each.
- For every `reject`, a **specific alternative** — usually adopting the benchmark as a *shape template* against your surface's distribution — with a pointer to the module in this track that owns the depth (`mod-103`, `mod-105`, `mod-108`, etc.). Do not reject without alternatives.
- One-line closer with a next step (e.g., "I can bring the adapted τ-bench-shape suite proposal to Thursday's review — a draft criteria delta is attached").

Aim for the tone of a memo a PM will read all the way through, not an academic exchange.

### 4. Shape-template plan (short)

For each `adapt` verdict — usually the middle candidate — produce a compact "shape-template adoption plan":

- **What is kept from the public benchmark** (task format, metric family, reporting shape).
- **What is replaced** (inputs from your production distribution, reference answers or rubric grounded in your business context).
- **Where it lives** (which offline / online split from exercise-02; which module in this track owns the runner).
- **Fixture-hash and version discipline** (see `mod-106`).

## Starter guidance

- Do the reading first. Two paragraphs about a benchmark you have not read is not an audit; it is an opinion.
- Be honest about "unknown" cells — they are often the load-bearing evidence for `reject`.
- The temptation is to write "reject because it is not our distribution" for every candidate. Fight it: pick your three so at least one is genuinely arguable.
- If a candidate is a *methodology* rather than a task benchmark (RAGAS is closer to a methodology library), your verdict may be `adopt as scoring stack` rather than `adopt as benchmark`. That is fine — say so.

## Acceptance criteria

You are done when:

- Three benchmarks are audited across all eight questions with full-sentence answers.
- Each has a clear `accept` / `adapt` / `reject` verdict with defensible rationale.
- The stakeholder memo is ≤ 300 words, includes at least one specific alternative for every `reject`, and reads as something you could actually send.
- Every `adapt` verdict has a compact shape-template plan.
- The whole file makes clear what the surface-relevant question was, how each benchmark answered or failed to answer it, and what the plan is going forward.

## Stretch goals

- **A/B interface note.** For one of your accepts or adapts, sketch how the resulting number would interface with an A/B experiment for a model bump on this surface — noting that A/B experimental methodology depth is owned by `model-evaluation-engineer` / `senior-ml-engineer`.
- **Recompute risk.** For the accepted / adapted benchmark, estimate what fraction of its score is likely dominated by prompt-template choice, referencing chapter 05's failure-mode list. Where would you want `model-evaluation-engineer` to double-check?

## What this exercise does *not* cover

You are not running the benchmark yet (that is `mod-103` / `mod-105` / `mod-108` depending on shape), and you are not adjudicating a model choice on its result (`mod-107`, `mod-111`, and the peer `model-evaluation-engineer` track own the depth). You are practising the *auditing* skill that gates whether a benchmark should be run at all.
