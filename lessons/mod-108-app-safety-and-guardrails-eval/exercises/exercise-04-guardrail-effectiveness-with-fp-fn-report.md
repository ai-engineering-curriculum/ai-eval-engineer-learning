# exercise-04: Guardrail Effectiveness With FP / FN Report

**Estimated effort:** 3 hours

## Objective

Score at least **two guardrails** — one OSS (Llama Guard 3, NeMo Guardrails, or Guardrails AI) and one hosted (OpenAI Moderation, Azure AI Content Safety, or Google Model Armor) — on a **policy-aligned labelled set**, and publish a per-category **confusion-matrix report** with explicit cost and latency accounting. The report must (a) show FP and FN separately, (b) name the operating point each number is measured at, (c) compare guardrails head-to-head plus the ensemble, and (d) produce a recommendation line the safety team can act on.

This exercise builds the guardrail axis of the app-safety scorecard. Exercise 05 stitches all four axes into an OWASP-mapped runbook.

## Prerequisites

- Chapters 01 and 05 of this module.
- Chapter 06 of this module (the labelled set is policy-aligned; if the policy reconciliation is not built yet, use a small policy-owner-labelled subset for this exercise and note the gap).
- A written safety policy document (chapter 06) — even a stub with 3 categories is enough to start.
- Provider accounts / API keys for the hosted guardrails you compare, each scoped to `EVAL_GUARDRAIL_*_KEY` (separate from production keys).
- Python 3.11+; the OSS guardrail installed (`llama-guard`, `nemoguardrails`, or `guardrails-ai`); the hosted-guardrail SDK you need.
- Optional: HuggingFace model download for Llama Guard 3 if running locally (~8 GB); or use it through a hosted inference endpoint.

## Set-up

1. Create `eval/safety/guardrails/` in your repo:

   ```
   eval/safety/guardrails/
   ├── config.yaml
   ├── labels/
   │   ├── labelled_set.jsonl   # policy-labelled rows; PII-scrubbed
   │   ├── policy_categories.md # your policy categories + example rows per category
   │   └── labeller_notes.md    # who labelled, when, what's ambiguous
   ├── guardrails/
   │   ├── llama_guard_3.py
   │   ├── openai_moderation.py
   │   ├── azure_content_safety.py    # or google_model_armor.py
   │   └── nemo_guardrails.py         # or guardrails_ai.py
   ├── scorer.py                # confusion-matrix computation + head-to-head
   ├── operating_point.py       # per-category threshold selection
   ├── runner.py
   ├── schema.py
   ├── budget.md
   ├── report/
   │   └── scorecard.md         # the end-of-run report
   ├── runbooks/
   │   ├── fpr_regression.md
   │   └── fn_regression.md
   └── tests/
       ├── test_scorer.py
       ├── test_operating_point.py
       └── test_label_freshness.py
   ```

2. In `config.yaml`, declare the guardrails to score, the labelled set, the operating-point search space, and the budget:

   ```yaml
   surface: support_bot
   labelled_set: labels/labelled_set.jsonl
   policy_snapshot: safety_policy_v11
   guardrails:
     - id: llama_guard_3_at_input
       module: eval.safety.guardrails.llama_guard_3
       config: {model: meta-llama/Llama-Guard-3-8B, position: input}
     - id: openai_moderation_at_output
       module: eval.safety.guardrails.openai_moderation
       config: {model: omni-moderation-latest, position: output}
   operating_point:
     search_space:
       threshold: [0.1, 0.15, 0.2, 0.25, 0.3, 0.35, 0.4, 0.5, 0.6, 0.7]
     per_category_search: true
     constraints:
       max_fpr_benign: 0.005
       min_recall_by_category:
         S8_self_harm: 0.99
         S1_violence: 0.90
         S3_sexual: 0.95
   budget:
     max_usd_per_run: 10.0
   ```

3. Build a *labelled set*. Requirements:
   - At least 200 rows total, stratified — roughly half harmful (per your policy), half benign, per each policy category you can label.
   - Include an **adversarial benign subset** (chapter 06 XSTest / OR-Bench shape) — prompts that superficially resemble unsafe categories but are policy-allowed. This is where over-refusal shows.
   - PII-scrubbed; no real user turns in the clear (chapter 02's payload contract).
   - Every row has `policy_label`, `policy_category`, `label_source` (human labeller id or role), `label_date`. Labels are the policy owner's or a delegate's, not an LLM alone.
   - Rows are versioned; `labels/labeller_notes.md` records ambiguous rows and the resolution.

## Requirements

Produce a PR against your working branch that adds:

1. **`labels/labelled_set.jsonl`** — the labelled set as above. Include a `README` header in the file documenting the labelling process and the policy snapshot the labels were made against.
2. **`labels/policy_categories.md`** — the categories your policy defines; a brief description per category; the disposition per category (per chapter 06).
3. **`labels/labeller_notes.md`** — the ambiguous rows and the resolutions; the labellers and the dates.
4. **`guardrails/<name>.py`** for each guardrail scored — a thin wrapper exposing a common interface:

   ```python
   class GuardrailAdapter:
       def score(self, text: str, position: str) -> GuardrailScore:
           # returns {categories: {S1: 0.02, S8: 0.71, ...}, latency_ms: float, cost_usd: float}
   ```

5. **`operating_point.py`** — given per-guardrail scores on the labelled set and the config's `constraints`, searches the threshold space for the operating point that satisfies the constraints. Supports per-category thresholds. Returns the point and the reason ("could not satisfy `min_recall S8_self_harm 0.99`; deployed the best available at 0.985").
6. **`scorer.py`** — computes the per-category confusion matrix (TP, FP, FN, TN, precision, recall, FPR, F1) plus the head-to-head table and the ensemble row. Renders the scorecard.md report.
7. **`runner.py`** — iterates the labelled set × guardrails; scores each row per guardrail; writes scored rows to the mod-107 store; renders `report/scorecard.md`. `--dry-run` runs one row per guardrail without cost.
8. **`schema.py`** — extends mod-107 chapter 02 with the chapter 05 additions (`eval_type=guardrail_effectiveness`, `guardrail_id`, `guardrail_config_hash`, `guardrail_position`, `operating_point`, `policy_label`, `policy_category`, `guardrail_decision`, `guardrail_score`, `guardrail_categories_fired`, `verdict` in `{TP, FP, FN, TN}`, `guardrail_cost_usd`, `guardrail_latency_ms`, `human_reviewer_id`, `policy_snapshot`).
9. **`budget.md`** — cost derivation for the eval run *and* for a projected production deployment at your surface's request rate (chapter 05's `R × 86400 × Σ c_g × 30`).
10. **`report/scorecard.md`** — the end-of-run report. Includes: per-guardrail per-category confusion matrix; head-to-head table with ensemble row; operating-point choice and reason; cost and latency per guarded request; a recommendation line (`KEEP` / `REVERT` / `ESCALATE`) with the rationale.
11. **`runbooks/fpr_regression.md`** — the runbook for a rising FPR on a category: owner, threshold, first step (which guardrail moved? which threshold?), remediation options (raise threshold on this category, swap classifier, disable category), escalation to product if the operating-point choice is a product-level trade.
12. **`runbooks/fn_regression.md`** — analogous for a rising FN rate: owner, threshold, first step, remediation (lower threshold, add a second guardrail in ensemble, disable feature until fixed), escalation to `ai-risk-engineer` if the missed category indicates an unmodelled hazard.
13. **`tests/test_scorer.py`** — synthetic scored rows verify TP/FP/FN/TN counts; verify per-category precision/recall against hand-computed values.
14. **`tests/test_operating_point.py`** — synthetic score distributions verify the operating point respects the constraints; verifies "could not satisfy" is reported when constraints are impossible.
15. **`tests/test_label_freshness.py`** — asserts the labelled set's `label_date` is within `N` days for at least `X %` of rows; fails on staleness.
16. **A demonstration run** — score both guardrails on the labelled set end-to-end; commit `report/scorecard.md` with real numbers; include a mod-107-aggregator screenshot on the `guardrail=<name>` cohort.

## Starter guidance

- **The labelled set is the load-bearing input.** Two afternoons of policy-owner labelling time produces a better report than two weeks of guardrail integration on unlabelled rows. Start with 50 labelled rows per category, three categories; expand iteratively.
- **Do not use a vendor's provided eval set.** The vendor's labels align with the vendor's classifier; your report will look better than it is. Label against *your* policy.
- **Per-category thresholds are usually right.** Llama Guard 3, Azure AI Content Safety, and NeMo Guardrails all support per-category configuration; use it. A single global threshold under-serves the categories where recall matters most.
- **Report cost and latency together with FP / FN.** The recommendation line reads all four; a report that shows only FP / FN is missing what product needs to make the call.
- **The ensemble row is worth computing even if you do not ship the ensemble.** It shows the ceiling — how much FN reduction the second guardrail *could* buy, at what FPR / cost / latency price. Sometimes the answer is "not worth it"; the number gives product the choice.
- **Latency measurement is p50 and p95.** A p50 = 40ms guardrail with a p95 = 800ms tail is different from a p50 = 60ms guardrail with a p95 = 90ms tail; the second one is friendlier to the user-facing SLO.
- **Do not skip the runbooks.** These are the runbooks that fire on the online-loop drift-monitor alerts from mod-107; a runbook that punts on remediation is not defensible.
- **Test-drive the operating point in a staging replay.** Point the runner at a small replay of production traffic (from mod-107's scored-row store) at the chosen operating point; confirm the resulting FPR / FN rates match the report's projections within a small margin.

## Acceptance criteria

You are done when:

- The labelled set has at least 200 rows across at least 3 policy categories with human labels against the current policy snapshot; PII is scrubbed; `label_source` and `label_date` are populated.
- Both guardrail adapters implement the common interface and score the labelled set end-to-end.
- `report/scorecard.md` renders the per-category confusion matrix, the head-to-head table with ensemble row, the operating-point choice with reason, the cost per guarded request, the p50 / p95 latency, and a `KEEP` / `REVERT` / `ESCALATE` recommendation line.
- The mod-107 aggregator reads the scored rows without modification on the `guardrail_id` cohort.
- `test_scorer.py`, `test_operating_point.py`, and `test_label_freshness.py` pass in CI.
- `budget.md` shows both the eval run cost and the projected production deployment cost at your surface's request rate.
- Both runbooks name owner, threshold, first step, remediation, and escalation.
- A short `README.md` in `eval/safety/guardrails/` explains how to add a new guardrail, how to refresh the labelled set, and how to interpret the scorecard.

## Stretch goals

- **Third guardrail.** Add a third — e.g., a programmable rail (NeMo Guardrails Colang policy, Guardrails AI validator suite) plus the two you started with. Report the ensemble ceiling for three-way blocks.
- **ROC / PR curves per guardrail.** Plot the ROC and PR curves on the labelled set; commit the plots to `report/`. Useful for cross-vendor comparison at future operating points.
- **Category-mapping table.** Build a table mapping your policy categories to each guardrail's native categories (Azure AI Content Safety splits by severity; OpenAI Moderation has its own set; Llama Guard 3 uses MLCommons taxonomy). Feed this mapping into the ensemble decision (a category-specific ensemble only fires when the mapped vendor category is checked).
- **Production-replay validation.** Pull a sample of production rows from mod-107 (chapter 02's schema); score them at the chosen operating point; report the projected FP / FN and cross-check against user complaints / support tickets from the same period.
- **Continuous re-labelling loop.** Wire a small workflow: production rows flagged by the guardrail land in a labelling queue; a policy-owner delegate labels them; labels flow back into the labelled set on a monthly cadence. Feeds the mod-107 drift monitor's ground-truth stream.
- **Cost tracking on the online loop.** Wire the `guardrail_cost_usd` on scored rows to a mod-107 aggregator cohort so the daily / monthly guardrail-cost line item is a first-class metric alongside quality. Chapter 06 of mod-111 reads this.

## What this exercise does *not* cover

You are not authoring the safety policy (chapter 06 walks the shape; the policy itself is a product / legal artefact) or building the OWASP-mapped runbook (exercise 05). You are shipping the guardrail effectiveness axis of the app-safety scorecard.
