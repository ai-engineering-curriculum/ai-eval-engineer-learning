# mod-110-eval-data-platform-slice: Eval-Data-Platform Slice for AI Applications

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Design a trace warehouse + eval-result store schema with traceable lineage (model / prompt / chain / retriever / decoding config / seed / eval-set hash)
- Design a test-case management layer that lets product teams contribute, tag, and version test cases without touching the harness code
- Orchestrate multiple eval runners behind one plane: Promptfoo, DeepEval, RAGAS, Braintrust, W&B Weave, Langfuse, and custom evaluators
- Wire the eval platform into CI and into the online-eval loop with rate-limits, retries, and cost accounting per team
- Instrument the eval platform itself with SLO / SLA (eval-run latency, success rate, cost per team) so it's operable — not just a script
- Coordinate with `model-evaluation-engineer` when the org needs cross-modality / multi-tenant eval-as-a-service depth beyond a product-eval slice

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
