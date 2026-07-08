# mod-103-trajectory-and-tool-eval: Trajectory and Tool-Call Evaluation for Production Agents

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Design a trajectory-level scorer for a real product agent: per-step tool-call correctness (name, arguments, ordering), final-answer correctness, and partial-credit rubric
- Track and enforce budgets: cost per trajectory, latency per step, step count, retry rate — and gate a rollout on them
- Use Inspect (UK AISI) agent harness solvers/scorers/tools to author a product-shaped agent eval
- Adopt SWE-bench Verified, τ-bench, WebArena, GAIA, AgentBench, Berkeley Function-Calling Leaderboard, and ToolBench as *shape templates* for internal agent suites — without confusing them with the product-agent eval itself
- Design deterministic replay for failing trajectories (recorded tool responses, seeded randomness, pinned model version) and a post-mortem workflow that uses it

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
