# mod-111-cost-latency-quality-tradeoff: Cost / Latency / Quality Trade-off Evaluation

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 12 hours

## Learning objectives

- Design a multi-objective eval report (quality vs. cost vs. latency vs. safety) that lets product decide what to ship
- Evaluate model routing (frontier vs. mid-tier vs. open-source, or classifier-gated routing) with per-cohort quality preserved
- Run distillation / replacement regression tests: swap a cheaper model in and prove where quality moves
- Measure app-altitude latency the way users experience it (TTFT, TPOT, streaming p50/p95, retry latency) and reconcile with the MLPerf-shaped serving benchmarks owned by `model-evaluation-engineer` and `ai-infra-performance`
- Track token accounting per feature so an eval-driven cost report can drive product decisions

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
