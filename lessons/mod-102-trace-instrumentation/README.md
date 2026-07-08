# mod-102-trace-instrumentation: Trace-Level Instrumentation for LLM Apps and Agents

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Instrument an LLM application (chain + tool calls + RAG retrieval + downstream services) with OpenTelemetry GenAI / OpenInference semantic conventions
- Wire the traces through Arize Phoenix, Langfuse, W&B Weave, Braintrust, and LangSmith end-to-end and reason about the trade-offs
- Design a span-attribute schema that captures prompt / response / model version / tool call args / retrieved context / tokens / cost / latency / user-id (or hashed) at the granularity a debugger needs
- Build a trace-viewer workflow that reconstructs the full call tree of a nested agent and lets an engineer bisect a bad run
- Adopt a sampling policy that keeps trace cost inside a product budget (head sampling, tail sampling on error/quality signal, PII scrubbing)

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
