# exercise-02: Position, Length, and Self-Preference Bias Controls in the Judge Pipeline

**Estimated effort:** 3 hours

## Objective

Take the pairwise rubric from exercise-01 and run it against three bias controls: **swap-and-average** for position bias, **length-adjusted scoring** for length / verbosity bias, and a **cross-family diagnostic** for self-preference. Produce a small report showing what the raw judge would have said, what each mitigation changes, and what the residual bias looks like when you inspect the numbers side-by-side.

By the end you will have a working judge-runtime wrapper that swaps candidate order, records candidate lengths on the trace, and can produce a length-adjusted score; a diagnostic dashboard (or notebook) showing position-bias flag rate, length correlation, and cross-family agreement; and a written 400–500-word note on what each mitigation caught on your surface and what it did not.

## Prerequisites

- Chapters 01 and 02 of this module.
- Exercise-01 completed — you need a real pairwise rubric.
- A model API key or two (frontier + one from a different family for the self-preference diagnostic). If you cannot access two frontier families, an open-source model behind vLLM (chapter 03) counts as the second family.
- A small dataset of pairs: ~30 (input, candidate A, candidate B) triples from your surface, or synthesised by running your surface with two prompt variants. Keep the dataset in a CSV or JSONL you can load into a notebook.
- Python 3.11+, `scipy`, and whichever HTTP client / SDK you use for the judge model.

## Set-up

1. Load the pairwise rubric from exercise-01 into a prompt template that renders a G-Eval-style judge prompt. Do not add any bias controls to the prompt — the whole point is to control them at pipeline time, not prompt time. Note the prompt hash somewhere; you will attach it to every judgement.
2. Prepare the ~30 pair dataset. Make sure at least one third of pairs have **materially different candidate lengths** (e.g., a 3× length ratio); this is what makes the length-bias signal visible.
3. Prepare the trace / logging plumbing so every judgement records: candidate A length, candidate B length, position order, judge model, judge prompt hash, raw winner, raw rationale.

## Requirements

Produce a single directory `mod-104/exercise-02/` in your working repo containing:

1. **`judge_runtime.py`** (or equivalent) — a small library implementing:
   - `pairwise(candidate_a, candidate_b, rubric_prompt, judge_fn)` that runs the judge **twice with swapped order** and returns `{"winner": "A"|"B"|"tie", "position_bias_flag": bool, "rationales": [str, str]}` per chapter 02's aggregation rule.
   - `absolute(candidate, rubric_prompt, judge_fn)` that returns `{"score": ..., "rationale": ..., "candidate_len": int}`.
   - `length_adjust(scores_df)` that takes a DataFrame of raw judgements + candidate lengths and returns a length-adjusted score using a simple regression (chapter 02 cites the AlpacaEval length-controlled paper; a logistic regression of win probability on length difference is a fine implementation).
   - A trace-emission hook — real or fake — that logs the mod-102 EVALUATOR-span attributes for each call (`app.judge_tier`, `app.judge_prompt_hash`, `app.judge_swap_order`, `app.judge_position_bias_flag`, `app.judge_candidate_a_len`, `app.judge_candidate_b_len`).
2. **`run_pipeline.ipynb`** or `run_pipeline.py` — runs `pairwise` and `absolute` over the ~30-pair dataset with your primary judge, then repeats with a **different-family judge** for the cross-family diagnostic. Persists results.
3. **`diagnostics.md`** — a short written report (400–500 words) with three subsections:
   - **Position bias.** The position-bias flag rate on the raw run vs the swap-and-average run. How many judgements collapsed to `tie` under the mitigation. Any pair where the two orders flatly disagreed on winner — call these out with the two rationales.
   - **Length bias.** The Spearman correlation of raw judge score against candidate length (for absolute) and the win-rate of the longer candidate (for pairwise) on the raw run. The same numbers after length adjustment. Report whether length was materially driving the raw scores.
   - **Self-preference.** The agreement rate between the primary judge and the second-family judge on the swap-and-averaged pairs. Any pair where the two families disagreed — call these out. Do the disagreements correlate with which candidate came from which generator family? That is the self-preference signal.
4. **`residual-note.md`** — a 150–250-word note honestly stating what these three mitigations *did not* fix on your data. Candidate directions: "length bias re-emerges above 5x length ratio," "swap-and-average tie rate is 40% — the rubric is under-specified," "cross-family agreement is 60% — a stronger mitigation (ensemble judge, human calibration) is needed for release-gate work." Do not paper over a bad result — the honest residual is a first-class deliverable.

## Starter guidance

- **Do not implement the mitigations in the judge prompt** ("please ignore position...", "please treat both candidates equally..."). Zheng et al. 2023 showed prose instructions do not remove position bias. Implement swap-and-average at the pipeline layer.
- The **swap-and-average tie rate** is a diagnostic, not a bug. A tie rate near 0 on genuine pair data suggests the judge is being asked to score something too different (or the position bias is being over-mitigated by an over-strong tie rule). A tie rate that is very high suggests either the candidates are near-tied on quality or the rubric anchors from exercise-01 need to be sharpened.
- **Length adjustment is post-hoc**, not runtime. Judge with the raw pipeline; store candidate lengths; run the regression at report time. Do not truncate candidates before judging.
- For the **cross-family diagnostic**, budget ~30 pairs × 2 judges × 2 swap orders ≈ 120 calls total. Keep temperatures low; keep max_tokens small on the judge prompt. This is not an expensive experiment.
- **Save the raw judgements**, not just the aggregates. When your calibration in exercise-04 later disagrees with a specific pair, you will want to open the two rationales and audit.
- Use `scipy.stats.spearmanr` for the length correlation and `scipy.stats.bootstrap` for confidence intervals if you want them.

## Acceptance criteria

You are done when:

- `judge_runtime.py` exposes `pairwise` and `absolute` entry points, and `pairwise` runs the judge in both orders and applies chapter 02's aggregation rule.
- Every judgement in the run logs the six trace attributes named in the requirements.
- `diagnostics.md` reports concrete numbers (with denominators) for position-bias flag rate, length correlation / longer-candidate-win-rate, and cross-family agreement rate — before and after each mitigation.
- At least one pair is called out in each of the three diagnostic subsections with the raw rationales, so a reviewer can inspect what the bias looked like.
- `residual-note.md` names at least one residual failure mode the three mitigations did not solve on your surface, honestly.

## Stretch goals

- Add a **triangulation judge** — a third judge from a third family — and repeat the cross-family diagnostic as three-way agreement. Note whether three-way agreement is materially rarer than two-way.
- Implement **ensemble aggregation**: run the two families on the same pair, treat majority (or unanimity) as the outcome, and report the ensemble tie / disagreement rate. Cost implication: at least 4× per pair. Note where you would spend the budget from chapter 05's tier routing.
- Add a **rubric variant with an explicit length constraint** in the anchors (per chapter 02's dual mitigation). Re-run and compare the length correlation between the two rubric versions. Did the anchor-level constraint materially reduce the correlation?
- Attach the raw judgements to a real mod-102 trace backend (Phoenix, Langfuse, Braintrust, Weave, or LangSmith — whichever your team uses). Confirm the trace-viewer surfaces the swap order and position-bias flag on the EVALUATOR span.

## What this exercise does *not* cover

You are not calibrating against a human gold set (exercise-04). You are not choosing a framework beyond enough scaffolding to run the pipeline (exercise-03). You are not routing across judge tiers or building a drift monitor (exercise-05). Stay narrow: three biases, one dataset, one report.
