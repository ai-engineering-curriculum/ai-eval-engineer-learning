# Designing Absolute and Pairwise Rubrics for Real Product Surfaces

## Motivation

An LLM-as-judge run without a rubric is a vibe check. It will produce a number, that number will look plausible, and it will move for reasons nobody can defend to a stakeholder — a slightly warmer prompt, a slightly longer completion, a judge model version bump. The number is not measuring what you think it is measuring, and the release-gate decision you build on top of it is not defensible.

The rubric is the difference between a judge that answers a question and a judge that produces theatre. This chapter walks the two rubric families a product-shaped judge actually uses — **absolute** (score one answer on a scale) and **pairwise** (choose the better of two, or declare a tie) — and shows how to anchor each one against real evidence from the trace. The bias controls in chapter 02, the framework choices in chapter 03, the human-gold calibration in chapter 04, and the tier routing in chapter 05 all assume a rubric of the shape this chapter designs. Skip this and everything downstream is being calibrated against noise.

## Core concepts

### Absolute vs pairwise: what each is for

There are two shapes an LLM-as-judge task takes in practice, and they answer different product questions.

- **Absolute (single-answer) scoring.** Given one output — a support-agent reply, a meeting summary, a code review comment — return a score on a defined scale for a defined criterion. Example: *"Score this reply's groundedness against the retrieved documents on 1–5."* This is what you want when you need a **per-request SLI** you can average, threshold in a CI gate, or track over time. The absolute score attaches back to the trace (see mod-102's `EVALUATOR` span) and rolls up into the mod-101 SLIs.
- **Pairwise (A vs B) preference.** Given two candidate outputs to the same input, choose which is better on a defined criterion — or declare a tie. This is what you want when you are **comparing model versions, prompt variants, or tool choices** and the absolute scale is noisier than the difference you are trying to measure. Pairwise is the shape LMSYS Chatbot Arena uses (<https://lmsys.org/blog/2023-05-03-arena/>) and the shape the MT-Bench single-vs-pair discussion in Zheng et al. 2023 (<https://arxiv.org/abs/2306.05685>) unpacks in detail.

The two are not interchangeable. Absolute gives you a value you can put on a dashboard; pairwise gives you a comparison you can gate a release on. Most product surfaces need both — an absolute score for the online SLI, a pairwise judge for the offline A/B and release-gate work.

Rules of thumb:

- **Prefer pairwise for release-gate A/B**, especially when the difference between candidates is small compared to the scale noise. Pairwise sidesteps the "what does 3 vs 4 actually mean" argument.
- **Prefer absolute for continuous online monitoring**, because pairwise requires a baseline to compare against and that baseline drifts.
- **Never mix the two on the same axis in the same run** without a stated conversion rule (Elo, Bradley-Terry) — one row of your criteria table produces one shape of judgement.

### The anatomy of an absolute rubric

An absolute rubric that survives contact with production has six parts. Each part exists to remove a specific class of ambiguity.

1. **Criterion name and one-line definition.** Not "quality" — that is not a criterion. Instead: *"Groundedness: the reply's factual claims are supported by the retrieved documents."* The definition sits at the top of the judge prompt and is what the judge is instructed to score against.
2. **Scale and levels.** State the numeric range and the meaning of every integer. A 1–5 scale needs five sentences, not just endpoints. See "anchors" below.
3. **Anchors per level.** Every level of the scale has a written description with observable behaviour — "the reply cites at least one document by id and every factual claim maps to a retrieved chunk" — not "excellent". Anchors are the single biggest lever on inter-rater agreement (chapter 04) and on judge reproducibility across model versions.
4. **Evidence pointer.** What the judge is instructed to compare against — the retrieved documents on the RETRIEVER span, the tool return, a reference answer, the user's message, etc. If the rubric asks the judge to score against context the judge does not have, the score will be a hallucination.
5. **Tie/abstain policy.** State what the judge does when the input is out of scope, the evidence is missing, or the criterion does not apply. The default is a dedicated `not_applicable` value; forcing a real score in these cases contaminates the aggregate.
6. **Output schema.** The exact JSON (or other structured) shape the judge must return. A rubric that returns free text is not machine-consumable and cannot be attached back to the trace. Include a `rationale` string field — it is what mod-109's human reviewers read when they audit a judgement.

A worked absolute rubric for a support-agent reply's groundedness might look like:

```
Criterion: Groundedness
Definition: The reply's factual claims are supported by the retrieved documents.
Scale: 1–5.
Anchors:
  5 = Every factual claim in the reply is directly supported by a
      retrieved document and no unsupported claims appear.
  4 = All load-bearing claims are supported; peripheral claims may
      be paraphrased from general knowledge without contradiction.
  3 = At least one load-bearing claim is unsupported or paraphrased
      without a clear source; no direct contradictions.
  2 = At least one load-bearing claim contradicts a retrieved
      document, OR the reply invents a specific fact (name, number,
      date) that is not present in retrieval.
  1 = The reply is broadly disconnected from retrieval; multiple
      unsupported specific claims or a contradiction of a
      user-visible fact.
Evidence: retriever.docs.search span, retrieval.documents.*.document.content.
Not-applicable: reply contains no factual claims (greeting, refusal).
Output: {"score": 1|2|3|4|5|"not_applicable", "rationale": "<=200 chars"}.
```

This is the shape the judge prompt renders — verbatim in the prompt, verbatim in the schema — so a reader can trace an on-call complaint from the number back to the level definition without opening the judge model.

### The anatomy of a pairwise rubric

Pairwise removes the absolute-scale ambiguity but adds two new failure modes: **position bias** (Zheng et al. 2023 show that judges systematically prefer position A) and **tie handling**. Chapter 02 handles position bias in the pipeline; the rubric handles the rest.

A pairwise rubric has five parts:

1. **Criterion name and one-line definition.** As in absolute.
2. **Comparison instruction.** Explicit: "Which reply better satisfies the criterion, given the shared input and shared retrieved context." State that the two replies were generated for the same input — the judge should not re-read the input as a spec ambiguity signal.
3. **Tie policy.** Ties are allowed *iff* the two replies are indistinguishable on the criterion. Force a choice when a subtle difference exists — many pairwise setups over-tie because the rubric does not push the judge to look for small differences. State the threshold: "call a tie only if you cannot state one concrete difference that would make one better than the other on this criterion."
4. **Evidence pointer.** Same as absolute — retrieved documents, tool returns, reference answer, etc. Pairwise judges hallucinate less when the evidence is in the prompt.
5. **Output schema.** Winner label plus rationale. Reserve a dedicated label vocabulary — `A`, `B`, `tie` — and do not accept anything else. Log the position each candidate occupied (chapter 02 needs this for the swap-and-average).

Worked pairwise rubric for reply groundedness:

```
Criterion: Groundedness (pairwise)
Definition: Which reply better supports its factual claims with
  the retrieved documents.
Comparison: The two replies were generated for the same user
  message and against the same retrieved documents. Judge only
  groundedness, not style or length.
Tie policy: Call a tie only if you cannot state one concrete
  difference that would make one better on groundedness.
Evidence: retriever.docs.search retrieval.documents.
Output: {"winner": "A"|"B"|"tie", "rationale": "<=200 chars"}.
```

The rubric prompt template must render `A` and `B` in a fixed order and log which candidate occupied which slot. Chapter 02 then runs the swap-and-average control that keeps position bias from moving your release-gate.

### Anchor design: the single largest lever

If you get anchors wrong, no amount of prompt-hardening or judge-model upgrade will save the judgement. The three failure modes to avoid:

- **Vague level descriptions ("good", "excellent")** — different judges land on different scores because "excellent" resolves to different behaviours per judge. Fix: every anchor states observable properties of the output ("cites at least one document", "no unsupported specific fact").
- **Non-monotone levels** — level 4 lists a criterion that level 3 does not mention, and level 5 lists yet another. The judge cannot ladder the response through the levels. Fix: define the levels as a *chain of relaxations* from 5 downward — each lower level relaxes one property from the level above.
- **Anchors that reference unavailable evidence** — the anchor says "the reply is faithful to the source documents" but the judge prompt does not include the source documents. Fix: every anchor references only evidence the judge prompt materialises.

Test the anchors on ~10 real traces before you ship: read each output, decide by hand which anchor it hits, and mark disagreements. If two humans on your team disagree on the anchor for a given trace, the anchor is under-specified. Rewrite before running the judge.

### Ground the rubric in a product criterion, not a benchmark family

A common failure mode is authoring a "faithfulness" rubric that was designed for a public benchmark (TruthfulQA, MT-Bench category X) and dropping it into your product surface. The rubric will run; it will produce numbers; the numbers will not move on real bugs, because the rubric is scoring the wrong axis for your surface.

Anchor every rubric to a specific row of your mod-101 criteria table. If mod-101 says the surface must maintain "Response reflects only what's in the retrieved policy PDF," your groundedness rubric levels reference retrieval; if it says "Reply cites the paragraph number when a policy question is asked," a level says so. The mod-101 criteria row is the design brief; the rubric is the implementation.

### Rubrics for reasoning traces vs final outputs

For agent-shaped surfaces, you often need rubrics that judge the **trajectory** (the tool calls, the intermediate reasoning) separately from the **final output**. This module's chapter-05 tier routing and mod-103's trajectory eval are the two places these rubrics land. Rules of thumb:

- **Final-output rubric** — absolute or pairwise, scored on the completed reply. Cheap. Attach to the AGENT root span.
- **Trajectory rubric** — usually absolute, scored on the sequence of tool calls / retrieval / intermediate messages. Cheap only for short trajectories; can get expensive with long ones. Attach to the AGENT root span with an `app.trajectory_score` extension.

Both are legitimate criteria; keep them as separate rubrics with separate anchors — do not average trajectory quality into output quality.

### What the rubric prompt template looks like end-to-end

A production-ready absolute-rubric prompt template has these sections, in order, before the response format instruction:

1. **Role sentence.** "You are an evaluator. Read the reply and the retrieved documents. Return only JSON."
2. **Criterion block.** The name, definition, scale, and anchors verbatim from the rubric.
3. **Evidence block.** The user message, the retrieved documents, and any tool returns, clearly delimited.
4. **Candidate block.** The output being scored.
5. **Instruction to reason briefly, then produce JSON.** G-Eval's chain-of-thought pattern (chapter 03) applies here.
6. **Response format spec.** Exact JSON schema.

The prompt template gets versioned in the same repo as the rubric, and its hash goes on every judgement (`app.judge_prompt_hash`) so mod-107's drift monitor can detect a rubric change. Chapter 05 covers the drift monitoring.

### What to write down before you write the judge

Before you write a judge prompt, write:

- The **criterion row** in the mod-101 criteria table this rubric implements.
- The **rubric document** with the six (absolute) or five (pairwise) parts from above.
- The **evidence contract** — the exact spans / attributes from mod-102 the rubric reads.
- The **acceptance test** — a small gold set (chapter 04) with hand-scored examples, one per anchor level for absolute, one per outcome (A / B / tie) for pairwise, minimum.

If any of the four is missing, do not run the judge. The output is meaningless without them, and any release decision built on it is a random walk.

## Summary

- Two rubric families: **absolute** for per-request SLIs and monitoring; **pairwise** for A/B and release-gate decisions. Do not mix on the same axis without a stated conversion rule.
- An absolute rubric has six parts: criterion + definition, scale, anchors per level, evidence pointer, tie/abstain policy, output schema. A pairwise rubric has five: criterion + definition, comparison instruction, tie policy, evidence pointer, output schema.
- Anchors are the largest lever on judge reproducibility. Every level must state observable properties, ladder monotonically, and only reference evidence the judge prompt materialises.
- Ground every rubric in a row of your mod-101 criteria table — not in a benchmark's rubric.
- Judge trajectories and final outputs with separate rubrics; do not average across.
- Version the rubric and the prompt template; hash the prompt onto every judgement so chapter-05 drift monitoring works.

Chapter 02 walks the position, length, and self-preference biases these rubrics are exposed to and the controls you build into the pipeline.
