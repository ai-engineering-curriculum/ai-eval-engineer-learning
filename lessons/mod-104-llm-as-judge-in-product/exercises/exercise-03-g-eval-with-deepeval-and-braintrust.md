# exercise-03: G-Eval with DeepEval and Braintrust (and a Framework Bake-Off)

**Estimated effort:** 3 hours

## Objective

Implement the **absolute** rubric from exercise-01 as a G-Eval-style chain-of-thought judge in **two** of the mainstream frameworks — **DeepEval** and **Braintrust** — and, as a shorter third comparison, wire the same rubric into **Promptfoo**'s `llm-rubric` assertion. Run all three against the same small dataset, and write a short bake-off note on which one you would keep for your surface and why.

By the end you will have three parallel implementations of the same rubric, a shared dataset, an aggregate agreement number for each framework against the others, and a written recommendation with the shape you would defend to a peer team. The output is what a mod-107 online-eval loop would consume.

## Prerequisites

- Chapters 01, 02, and 03 of this module.
- Exercise-01 completed — the absolute rubric.
- Exercise-02 completed — the pipeline runtime, which you will now delegate to a framework.
- Python 3.11+, `deepeval`, `braintrust`, `autoevals`, and the `promptfoo` CLI (Node.js). Install fresh; the frameworks move.
- A frontier judge API key. Use the same judge model across all three implementations for the primary run — the frameworks are what we are comparing, not the model.
- ~20 (input, retrieved context, candidate) triples from your surface — the same dataset shape as exercise-02, plus retrieved context if your rubric reads it.

## Set-up

1. Author the rubric prompt template *once*, in a plain-text file — `rubric.txt` — containing exactly the prompt each framework will render. This is the ground-truth prompt; framework-specific rendering is what you compare against it.
2. Configure each framework with the **same judge model, same temperature, same max_tokens.** The comparison is meaningful only when these are controlled. Confirm you can override them in each framework's configuration surface.
3. Prepare the dataset in a shape each framework accepts: for DeepEval, a list of `LLMTestCase`; for Braintrust, an Experiment dataset; for Promptfoo, a `tests:` block in `promptfooconfig.yaml`.

## Requirements

Produce a single directory `mod-104/exercise-03/` in your working repo containing:

1. **`rubric.txt`** — the prompt template rendered as plain text. This is the source of truth all three framework configurations render against.
2. **`deepeval_run.py`** — implements the rubric using DeepEval's `GEval` metric (docs: <https://docs.confident-ai.com/>). Consume the shared dataset; emit per-item scores and rationales to `results-deepeval.csv`. Configure `evaluation_steps` from the rubric anchors; keep `threshold` explicit rather than defaulting.
3. **`braintrust_run.py`** — implements the same rubric via Braintrust's `autoevals` library (docs: <https://www.braintrust.dev/docs>). Use the closest-fit grader (`ClosedQA` or `LLMClassifier` — pick per your rubric shape and defend the choice in the bake-off note). Emit per-item scores to `results-braintrust.csv`. Run under a Braintrust experiment so the traces are stored alongside the scores.
4. **`promptfooconfig.yaml`** — implements the rubric via Promptfoo's `llm-rubric` assertion (docs: <https://www.promptfoo.dev/docs/configuration/expected-outputs/model-graded/>). Emit per-item pass/fail to `results-promptfoo.csv` (Promptfoo's default output CSV works).
5. **`bakeoff.md`** — 500–700 words in three sections:
   - **Prompt-rendering diff.** For each framework, show the actual prompt the judge received (grab it from framework debug logs, framework docs on prompt rendering, or by intercepting the outbound API call). Where each framework injected extra scaffolding, reformatted anchors, or truncated. Confirm that the intent of `rubric.txt` survived each renderer.
   - **Score-agreement matrix.** A 3×3 matrix of pairwise agreement between the three frameworks' judgements on the same items (Spearman ρ for ordinal, or fraction-agreement for pass/fail). Read as: "how much does the framework choice move the score before you even change the judge model." A perfect matrix suggests all three frameworks render your rubric identically; a matrix with a bad row is a framework you should not use for this rubric.
   - **Recommendation.** Which framework you would keep for your surface, and why. Grounded in: how well it renders your rubric, how naturally it wires to your team's existing infra (pytest / trace backend / CI), how it handles judge swaps, and how it emits per-item results.
6. **`framework-tradeoffs.md`** — a short table (~10 rows) with the per-framework properties you would consult next time: pytest-native (yes/no), CI-runner-native (yes/no), auto-tracing (yes/no + which backend), judge model pluggability (yes/no + how), pairwise support (yes/no), G-Eval logit-weighted mode (yes/no), open-source (yes/no), license. Fill from the current docs at the linked URLs. Do not invent.

## Starter guidance

- **Author `rubric.txt` first, then port to each framework.** Do not tune per framework — the goal is a fair comparison. If a framework's renderer garbles the anchors, that is data for the bake-off note, not a call to rewrite the rubric.
- **Framework debug logging is your friend.** DeepEval, Braintrust, and Promptfoo can all be configured to log the exact judge prompt they emit; do this. The rendered prompt is what actually determines the score, not your `rubric.txt`.
- If your framework's built-in scorer does not fit your rubric shape (e.g., you want an ordinal 1–5 scale but the framework's default is pass/fail), **note the mismatch in `bakeoff.md`** and either configure the framework to match or use its lower-level judge callback. Do not silently coerce.
- Keep the dataset small — 20 items × 3 frameworks × 1 judge call each is enough to see divergence. This is a bake-off, not a benchmark.
- If the same judge model is not available on all three frameworks with identical settings, **document the delta** in the recommendation. Frameworks change; use current docs (linked in `framework-tradeoffs.md`) to configure identical judge settings where possible.

## Acceptance criteria

You are done when:

- `rubric.txt` is the single source of truth for the rubric prompt template; the three framework configurations render from it (with framework-specific scaffolding acknowledged and diffed in the bake-off note).
- All three implementations run end-to-end on the same 20-item dataset and produce per-item scores.
- `bakeoff.md` contains the prompt-rendering diff, the score-agreement matrix (real numbers with denominators), and a defensible recommendation grounded in your surface's existing infra.
- `framework-tradeoffs.md` is filled in from the current linked docs — no invented rows.
- Nothing in the repo hard-codes an API key; keys are read from environment variables.
- All three frameworks were configured with the **same judge model, temperature, and max_tokens** where possible; any delta is documented.

## Stretch goals

- **Add an open-source-judge variant.** Point one of the frameworks at a self-hosted Prometheus 2 or JudgeLM behind vLLM (chapter 03), using the same rubric. Compare the OSS score column against the frontier column in the score-agreement matrix. This is the exercise-05 tier-routing signal you will use later.
- **Add the pairwise rubric from exercise-01** to Braintrust's `Battle` (or the closest equivalent in DeepEval / Promptfoo). Confirm the framework applies chapter 02's swap-and-average control; if it does not, note that as a gap in `framework-tradeoffs.md`.
- **Wire the results to a real trace backend.** If your team runs Phoenix / Langfuse / LangSmith / Weave, configure the framework to emit judge scores as EVALUATOR spans on the corresponding traces. Confirm the trace-viewer surfaces the score + rationale on the span.
- **Turn on G-Eval's probability-weighted score mode** (DeepEval GEval's default) and compare against a run with it turned off. Report the score-distribution shift.

## What this exercise does *not* cover

You are not calibrating against a human gold set (exercise-04). You are not designing routing between judge tiers (exercise-05). You are not implementing bias controls beyond what each framework ships (exercise-02 built the controls). Stay narrow: the same rubric, three frameworks, one comparison.
