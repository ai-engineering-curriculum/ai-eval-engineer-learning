# mod-106-eval-gated-cicd: Eval-Gated CI/CD for Prompts, Chains, and Agents

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Wire evaluations into pull-request CI so a prompt / chain / agent change gets an eval report before merge
- Snapshot and replay production traces as regression fixtures with pinned model version, seed, and retrieval index hash
- Register pre-registered pass/fail thresholds and treat regressions as first-class blockers with an owner-assigned runbook
- Build declarative eval configs with Promptfoo (YAML), DeepEval pytest hooks, Braintrust / Weave / Langfuse CI runners
- Design deployment gates (offline-eval pass → canary → progressive rollout) and the interface to the deploy pipeline the platform team owns

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
