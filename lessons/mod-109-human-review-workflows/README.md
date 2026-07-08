# mod-109-human-review-workflows: Human Review Workflows for AI Product Teams

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 10 hours

## Learning objectives

- Stand up an expert-review workflow in Argilla or Label Studio and integrate it with the trace store
- Capture thumbs-up/down and structured feedback from real users, dedupe it, and route it into an evaluation dataset with owner-assigned triage
- Compute practical inter-annotator agreement (Cohen's / Fleiss' kappa) at product scale — enough to decide whether a rubric is stable, and know when to escalate to `model-evaluation-engineer` for methodology depth
- Rotate gold sets in step with product-surface changes so drift does not silently erode the offline suite
- Choose vendor vs. in-house review for a given surface with data-residency, latency, and cost trade-offs explicit — deferring vendor-methodology depth to `model-evaluation-engineer`

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
