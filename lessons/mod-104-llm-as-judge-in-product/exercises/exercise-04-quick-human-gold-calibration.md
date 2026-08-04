# exercise-04: Quick Human-Gold Calibration of a Judge

**Estimated effort:** 3 hours

## Objective

Build a small stratified human-labelled gold set for the absolute rubric from exercise-01, run the judge from exercise-03 on it, and produce a **calibration document** with an appropriate statistic (Cohen's κ, Spearman ρ, Kendall τ, or three-class κ + non-tie agreement, per rubric shape) plus a 95% bootstrap CI and a per-stratum breakdown.

The output is the artefact mod-107's online-eval loop points at when a stakeholder asks *"how do we know the judge score is right?"*. It is also the entry-point for the delegation contract with the **`model-evaluation-engineer` peer** (chapter 04) — the last question in this exercise is *"is what I have here defensible, or should I escalate?"*

By the end you will have a labelled CSV, a calibration script that computes the statistic with bootstrap CIs, a one-page calibration document per chapter 04's spec, and a note on where you would escalate to the peer role.

## Prerequisites

- Chapter 04 of this module.
- Exercise-01 (the rubric), exercise-03 (the framework wiring). Exercise-02's bias controls should be applied to the pipeline you calibrate.
- Python 3.11+, `pandas`, `scipy` (for `spearmanr`, `kendalltau`, `bootstrap`), and `sklearn.metrics` (for `cohen_kappa_score`).
- 50–100 recorded (input, retrieved context, candidate) triples from your surface, respecting mod-102's sampling and PII policy. If your surface is pre-launch, synthesise the traces from a fixture.
- A second labeller if possible (a teammate, a domain expert, yourself two days apart). If only one labeller is available, note this in the calibration document as a limitation.

## Set-up

1. Draw a **stratified sample of 50–100** items from your production trace store or fixture set. Stratify by:
   - Rubric outcome level (for absolute 1–5, roughly equal counts per level).
   - Surface sub-class (tool-call vs no-tool-call, topic category, first-turn vs follow-up).
   - Known failure modes if you have a bug taxonomy from mod-102's bisection or mod-109 labels.
   Log the stratification scheme — the calibration doc will consume it.
2. Prepare a labelling spreadsheet with the input, retrieved context, and candidate rendered clearly (not raw JSON blobs). One row per item. The labeller reads the rubric from exercise-01, not the judge prompt.
3. Have the labeller(s) score each item against the rubric. Do **not** show them the judge's score. Time-box; do not overthink individual items.
4. Run the judge (exercise-03) on the same items. Keep temperatures low; if the judge is non-deterministic, run twice and average or take the majority — note the choice.

## Requirements

Produce a single directory `mod-104/exercise-04/` in your working repo containing:

1. **`gold-set.csv`** — the labelled dataset. Columns: `item_id`, `stratum` (e.g., `level_5|tool_call|first_turn`), `input`, `context` (or a pointer to the trace), `candidate`, `human_score`, `human_labeller`, `human_labelled_at`. If two labellers, add `human_score_2`, `human_labeller_2`.
2. **`judge-run.csv`** — one row per item with `item_id`, `judge_model`, `judge_prompt_hash`, `judge_score`, `judge_rationale`, `judge_run_at`. If bootstrapped over multiple judge runs, columns for each.
3. **`calibrate.py`** — computes:
   - The appropriate statistic given your rubric shape (see chapter 04 table). Prefer `sklearn.metrics.cohen_kappa_score` for nominal, `scipy.stats.spearmanr` / `scipy.stats.kendalltau` for ordinal. For pairwise (if you calibrate the pairwise rubric instead), three-class Cohen's κ over `{A, B, tie}` and the non-tie agreement rate as a secondary.
   - A 95% bootstrap CI on the statistic (`scipy.stats.bootstrap`, 1000+ resamples).
   - A per-stratum breakdown — the same statistic computed per stratum, and a per-stratum count.
   - Inter-labeller agreement on a shared sub-sample if you have two labellers.
4. **`calibration.md`** — the one-to-two-page calibration document per chapter 04's eight-part shape:
   - Rubric and version (pointer + prompt hash).
   - Judge model + framework + version.
   - Gold set: size, source, stratification scheme, per-stratum counts.
   - Rater(s): who, when, inter-rater agreement if measured.
   - Statistic and value with 95% CI, plus per-stratum breakdown table.
   - Decision: is the judge considered calibrated? Against what threshold? Cite the SLI in mod-101 that consumes it.
   - Re-calibration triggers: what changes require re-calibration (judge upgrade, rubric edit, distribution shift).
   - Escalation status: has the model-evaluation-engineer peer been asked to produce full-methodology work; what interim number stands.
5. **`escalate-or-not.md`** — 200–300 words. Honestly answer: is what you have defensible for a release-gate decision at your surface's altitude, or should you escalate to the `model-evaluation-engineer` peer? Cite one or more of chapter 04's escalation triggers if you would escalate — high-stakes rubric needing variance decomposition, need for a statistical power analysis, ordinal scale with unequal-interval anchors, multi-rater aggregation, non-standard resampling. If you would not escalate, defend that — the honest "we are fine at this altitude" is a legitimate answer.

## Starter guidance

- **Do not read the judge's rationale before you label.** Human labellers are subtly nudged by the judge's phrasing. Judge the item against the rubric alone.
- **Time-box the labelling** — ~2 minutes per item. If you spend 10 minutes on an item, the rubric is under-specified for that item; either the anchors need work (exercise-01 loop) or the item's context is unavailable.
- **Show the judge and human results side-by-side in a scan pass** after the calibration is computed, and inspect the worst 5 disagreements. If they cluster on one stratum, the calibration document must call this out.
- **Report the 95% CI, not just the point estimate.** A κ = 0.55 with CI [0.30, 0.75] is not a defensible release-gate number even though the point estimate looks OK.
- If you have only one labeller, **note it as a limitation** and treat the calibration as provisional. That is legitimate at product altitude; chapter 04 is explicit that quick-calibration has bounded rigour.
- Use `scipy.stats.bootstrap` with `method="BCa"` for the CI if the data are small — it's a standard choice for small-N bootstrap. If your dataset is < 30, seriously consider re-scoping the exercise: a κ CI on 20 items is very wide.
- **Do not run the calibration over and over until it looks good.** If the number is bad, the fix is: rewrite anchors (exercise-01 loop), add bias controls (exercise-02), switch judge tier (exercise-03), or escalate (chapter 04).

## Acceptance criteria

You are done when:

- `gold-set.csv` has 50–100 stratified items with human scores, and stratification is documented.
- `judge-run.csv` has one row per gold-set item with score, rationale, judge model, and prompt hash.
- `calibrate.py` computes the correct statistic for your rubric shape, plus 95% bootstrap CI and per-stratum breakdown.
- `calibration.md` fills in all eight chapter-04 sections. The re-calibration-trigger list is concrete (not "we will re-calibrate when needed").
- `escalate-or-not.md` names a specific chapter-04 escalation trigger — either invoked (and the delegation contract is described) or explicitly declined and defended.
- The per-stratum breakdown is present in the calibration doc; the worst-stratum statistic is called out.
- If the aggregate κ / ρ / τ CI is wide enough to overlap the null or a threshold, the calibration doc explicitly says the judge is *not* considered calibrated at this time.

## Stretch goals

- **Extend the gold set from 100 to ~200** stratified items and re-run the calibration. Note how the CI narrows and whether the per-stratum picture changes. This is the boundary where escalation to peer becomes more defensible.
- **Add a second judge model** (the mid-tier judge from chapter 03) and produce a parallel calibration doc for it. Compare the two calibration numbers side-by-side — this is the input to exercise-05's tier routing.
- **Set up a scheduled re-calibration harness** — a script that pulls the gold set, runs the current-production judge, and re-computes the statistic weekly. Write the alert threshold that will fire when the statistic drifts. Wire it to whatever notification channel your team uses. This is the exercise-05 drift-monitor scaffold.
- **Draft the delegation contract** with the `model-evaluation-engineer` peer even if you would not escalate today. Ideal shape: what would you send them, what would you get back, what would you commit to using the returned analysis for. This is the mod-112 governance pattern in miniature.

## What this exercise does *not* cover

You are not designing tier routing or the drift monitor (exercise-05). You are not building the mod-107 online-eval loop that reads this calibration (mod-107's exercise). You are not writing the peer's full-methodology analysis — that is by design out of scope at product altitude. Stay narrow: one rubric, one judge, one stratified gold set, one defensible number.
