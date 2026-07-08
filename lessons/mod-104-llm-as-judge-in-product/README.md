# mod-104-llm-as-judge-in-product: LLM-as-Judge in Product Pipelines: Rubrics, Bias Controls, and Judge-Tier Routing

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Design absolute (single-answer) and pairwise rubrics for real product criteria with explicit scoring anchors
- Apply position-bias controls (swap-and-average), length-bias controls (length-normalised scoring), and self-preference diagnostics inside the judge pipeline
- Use G-Eval-style chain-of-thought rubrics with DeepEval / Braintrust / Promptfoo and know when to switch to a dedicated open-source judge (Prometheus 2 / JudgeLM) for cost or reproducibility
- Quick-calibrate a judge against a small human gold set with a defensible statistic and know when to escalate to the `model-evaluation-engineer` peer for full methodology depth (Cohen's kappa / Spearman / Kendall / bootstrap CIs)
- Route judgements across judge tiers (frontier API / mid-tier / open-source) inside a cost budget and monitor for judge drift when the judge model itself upgrades

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
