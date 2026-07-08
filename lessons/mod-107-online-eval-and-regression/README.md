# mod-107-online-eval-and-regression: Online Evaluation and Regression Detection at the App Layer

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Design a sampled-trace online-eval loop that scores live traffic on quality / safety / cost / latency without breaking the budget
- Detect drift at input (query distribution shift), output (response distribution shift), and judge level (model-judge upgrade drift, judge-prompt drift)
- Wire canary and shadow launches with per-cohort quality and safety gates
- Apply sequential testing / confidence sequences for continuous monitoring so the alert fires when it should and not before
- Interface with the A/B platform: know the CUPED and pre-registration contract, and know where the deep experimental methodology (owned by model-evaluation-engineer / senior-ml-engineer) begins
- Wire dashboards in Arize Phoenix, Langfuse, W&B Weave, or Braintrust that a product manager, an on-call engineer, and a safety reviewer can each read

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
