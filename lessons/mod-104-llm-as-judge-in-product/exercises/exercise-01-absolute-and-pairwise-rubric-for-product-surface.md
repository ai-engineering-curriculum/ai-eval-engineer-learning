# exercise-01: Absolute and Pairwise Rubrics for a Product Surface

**Estimated effort:** 3 hours

## Objective

Produce two production-ready judge rubrics — one **absolute** and one **pairwise** — against a real product surface's criteria. The rubrics are the design brief for every downstream exercise in this module: exercise-02 will subject them to bias controls, exercise-03 will implement them in a framework, exercise-04 will calibrate them against a small human gold set, and exercise-05 will route judgements across judge tiers using them. If the rubrics are vague here, everything downstream inherits the vagueness.

By the end you will have two rubric documents (definition, scale, anchors, evidence, tie/abstain, output schema for absolute; and comparison instruction + tie policy for pairwise), a small acceptance test showing the rubrics differentiate three hand-picked outputs correctly, and a note on where each rubric attaches to the trace shape from mod-102.

## Prerequisites

- Chapter 01 of this module.
- Your mod-101 criteria table for a surface. If you have not built one, pick one of the mod-101/exercise-01 surfaces (support agent, meeting summariser, coding assistant) and produce a minimal criteria table first.
- Familiarity with the mod-102 span-schema (chapter 04 of mod-102) — the rubrics reference span attributes as evidence.
- No model API calls are required for this exercise. This is a design exercise; the judge will run in exercise-03.

## Choose a surface and a criterion pair

Pick the same surface you used in mod-101/exercise-01. Pick **two criteria** from your criteria table — one that naturally fits an absolute rubric (e.g., groundedness, refusal-safety, helpfulness) and one that naturally fits a pairwise rubric (e.g., "candidate A vs candidate B on reply quality" for an A/B between two prompt templates). It is legitimate for both criteria to describe the same underlying property (e.g., groundedness both ways) — the point is to feel the difference between the two rubric shapes.

If your surface has no natural pairwise use case yet, invent one: assume you are considering swapping the current prompt template for a shorter one, and the pairwise rubric will decide the swap.

## Requirements

Produce a single directory `mod-104/exercise-01/` in your working repo containing:

1. **`rubric-absolute.md`** — the absolute rubric, with the six parts from chapter 01:
   - Criterion name and one-line definition (anchored to a specific row of your mod-101 criteria table — cite the row).
   - Scale and levels.
   - Anchors per level, each stating observable properties (never "excellent", "good"). Anchors ladder monotonically.
   - Evidence pointer — the exact mod-102 spans and attributes the rubric reads (e.g., `retriever.docs.search retrieval.documents.*.document.content`).
   - Tie/abstain policy naming the `not_applicable` shape.
   - Output schema (JSON) with `score`, `rationale`, and any per-anchor evidence fields you want the judge to fill.
2. **`rubric-pairwise.md`** — the pairwise rubric, with the five parts from chapter 01:
   - Criterion name and one-line definition (cite the criteria-table row).
   - Comparison instruction: explicit that the two candidates are on the same input, same retrieval; judge only the named criterion.
   - Tie policy: force a call unless no concrete difference can be stated on the criterion.
   - Evidence pointer (same shape as absolute).
   - Output schema with `winner` in {`A`, `B`, `tie`} and `rationale`.
3. **`acceptance-test.md`** — a short manual acceptance test:
   - Three hand-picked example outputs from your surface (real or realistic) for the absolute rubric. Score each one by hand against the rubric. Show your work. At least one output at the top of the scale, one in the middle, one at the bottom.
   - Two hand-picked candidate pairs for the pairwise rubric. Judge each by hand. At least one A-wins, one tie.
   - Note any anchor you had to re-read to make the judgement, and rewrite that anchor in-place. Do this until every judgement is unambiguous from the anchors alone.
4. **`trace-attachment.md`** — 100–200 words. Which mod-102 EVALUATOR-span attributes each rubric emits back onto the trace: `app.judge_rubric_name`, `app.judge_prompt_hash`, `app.judge_score`, `app.judge_rationale` (absolute); `app.judge_winner`, `app.judge_swap_order`, `app.judge_position_bias_flag`, `app.judge_candidate_a_len`, `app.judge_candidate_b_len` (pairwise). Name any extension attributes you need beyond mod-102's baseline.

## Starter guidance

- Write anchors that reference **observable behaviour**, not judgement words. Bad: "the reply is clear." Good: "every noun phrase in the reply either appears in the retrieved documents or is a paraphrase of a phrase that does."
- Write anchors as a **ladder**: level 5 has property P₁; level 4 relaxes P₁ to a weaker property; level 3 relaxes further; and so on. Do not introduce new criteria at lower levels — that produces incomparable scores.
- **Pick the same criterion for both rubrics initially** (e.g., groundedness) — the shape difference between the absolute and pairwise versions is the point of the exercise. You will see the ambiguity absolute rubrics carry and the position-bias exposure pairwise rubrics carry.
- **Do not import a public benchmark's rubric**. TruthfulQA, MT-Bench, and similar benchmarks have rubrics designed for their evaluation setting; on your surface they will score something adjacent-but-different. Author from your criteria table.
- Prefer **evidence-visible** rubrics: if the rubric asks the judge to score groundedness, the retrieved documents must be in the judge prompt template. Note in the evidence pointer exactly which fields of which spans the prompt will materialise.
- Cap the `rationale` field length in the output schema (e.g., 200 characters). Uncapped rationales pad judge cost and add nothing to consumability.

## Acceptance criteria

You are done when:

- `rubric-absolute.md` has all six parts filled in against a real criterion in your mod-101 criteria table. The criterion row is cited.
- `rubric-pairwise.md` has all five parts filled in.
- Every anchor in the absolute rubric describes observable properties; a colleague could read the rubric alone (without your commentary) and score an output.
- Anchors ladder monotonically — no level introduces a criterion the level above did not mention.
- The output schema for each rubric is a real JSON schema (or equivalent) with fixed value vocabularies (`winner` ∈ {A, B, tie}; `score` ∈ {1, 2, 3, 4, 5, not_applicable}).
- `acceptance-test.md` shows three hand-scored absolute examples spanning the scale and two hand-scored pairwise examples with at least one tie, and any ambiguous anchor has been rewritten.
- `trace-attachment.md` lists the EVALUATOR-span attributes each rubric will emit, using mod-102's naming.
- No rubric depends on evidence the judge prompt does not materialise. Confirm by reading each anchor and checking the evidence pointer.

## Stretch goals

- Add a **third rubric** — an absolute rubric for a *trajectory* criterion on the same surface (e.g., "did the agent call the right tools in the right order"). Note in the rubric doc which mod-102 spans the trajectory rubric reads (the sequence of TOOL / LLM / RETRIEVER children of the AGENT root).
- Draft the **rubric prompt template** that renders each rubric into a G-Eval-style chain-of-thought judge prompt (chapter 03). Do not run it yet — exercise-03 does. Version-control the template; note its hash placeholder in `trace-attachment.md`.
- Sketch the mod-109 review workflow the rubric feeds: if a judgement disagrees with a spot-check reviewer, what does the reviewer see, and what field on the rubric changes to close the gap.

## What this exercise does *not* cover

You are not running a judge — that is exercise-03. You are not applying bias controls — that is exercise-02. You are not calibrating against a human gold set — that is exercise-04. You are not routing across tiers — that is exercise-05. Stay narrow: two rubrics, hand-tested, evidence pointers spelled out.
