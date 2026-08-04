# mod-102 — Trace-Level Instrumentation for LLM Apps and Agents

**Estimated effort:** 14 hours

This is the second module of the AI Evaluation Engineer track. It hands the rest of the track its raw material: a trace. Everything downstream — trajectory eval (`mod-103`), judge scoring (`mod-104`), the RAG-triad (`mod-105`), CI gates (`mod-106`), the online-eval loop (`mod-107`), guardrail measurement (`mod-108`), human review (`mod-109`), the eval-data platform (`mod-110`), the cost / latency / quality trade-off model (`mod-111`), and the program-owner posture (`mod-112`) — consumes traces of the exact shape you design here. `mod-101` gave you the SLIs; this module gives you the artefact those SLIs are computed against.

## Learning objectives

- Instrument an LLM application (chain + tool calls + RAG retrieval + downstream services) with OpenTelemetry GenAI / OpenInference semantic conventions.
- Wire the traces through Arize Phoenix, Langfuse, W&B Weave, Braintrust, and LangSmith end-to-end and reason about the trade-offs.
- Design a span-attribute schema that captures prompt / response / model version / tool-call args / retrieved context / tokens / cost / latency / user-id (or hashed) at the granularity a debugger needs.
- Build a trace-viewer workflow that reconstructs the full call tree of a nested agent and lets an engineer bisect a bad run.
- Adopt a sampling policy that keeps trace cost inside a product budget (head sampling, tail sampling on error / quality signal, PII scrubbing).

## Lecture chapters

1. [Why trace-level instrumentation for LLM apps](01-why-trace-instrumentation.md) — trace / span / event vocabulary; the five debugger questions; how an LLM trace differs from a normal APM trace; a worked call tree.
2. [OpenTelemetry GenAI and OpenInference semantic conventions](02-otel-genai-and-openinference.md) — the two live conventions, where they overlap and diverge, per-kind attribute vocabularies, minimal manual instrumentation snippets, and the dual-convention mirror pattern.
3. [Wiring traces through Phoenix, Langfuse, Weave, Braintrust, and LangSmith](03-backend-bakeoff.md) — SDK-direct vs OTLP transports, per-backend trade-offs, the portable OTel Collector recipe, and a defensible bake-off procedure.
4. [Designing the span-attribute schema for LLM apps and agents](04-span-schema-design.md) — SLI-derived per-kind MUST / SHOULD tables, the nine cross-cutting root attributes, content-granularity rules, user-id hashing, content sizing, and schema versioning.
5. [Bisecting a bad run: the trace-viewer workflow for nested agents](05-bisecting-a-bad-run.md) — the six-step bisection, common failure signatures and where they resolve on the tree, sub-trace / span-link handling, and the annotate-and-hand-off loop.
6. [Sampling policy and PII scrubbing at trace budget](06-sampling-and-pii-policy.md) — head / tail / signal-driven sampling, the two-layer scrubbing pattern, retention tiers and right-to-erasure, an OTel Collector recipe, budget math, and the residual-risk statement.

## Exercises

Each exercise builds on the last. Do them in order and keep the outputs — the artefacts (instrumented service, bake-off scorecard, schema doc, bisection report, sampling policy) are the deliverables `mod-106` / `mod-107` / `mod-110` will assume you have.

1. [OTel GenAI / OpenInference semantic conventions](exercises/exercise-01-otel-genai-semantic-conventions.md) — instrument a small chat-plus-tool service against both conventions and reason about which one you would keep.
2. [Phoenix / Langfuse / Weave / Braintrust / LangSmith bake-off](exercises/exercise-02-phoenix-langfuse-weave-bakeoff.md) — fan out via an OTel Collector to two backends, drill the debugger workflow, and write a defensible pick-a-backend memo.
3. [Span-schema design for agents](exercises/exercise-03-span-schema-design-for-agents.md) — turn your `mod-101` criteria table into a per-kind attribute schema with MUST / SHOULD tables, versioning, and an SLI-mapping table.
4. [Nested-agent trace-tree walkthrough](exercises/exercise-04-nested-agent-trace-tree-walkthrough.md) — bisect three bad runs against your instrumented app; produce annotations, reproducer fixtures, and mod-109 hand-off tickets.
5. [Sampling and PII policy](exercises/exercise-05-sampling-and-pii-policy.md) — write and validate the surface's sampling + scrubbing policy, budget math, SLI-preservation check, and residual-risk statement.

## Labs and quizzes

- Labs (see [`labs/`](labs)) build the end-to-end instrumented reference app that `mod-106` wires into CI and `mod-107` runs the sampled-eval loop against. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary and the six-step bisection. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- Trajectory / tool-call evaluation, driven by the AGENT / LLM / TOOL span sequence you emit here → [`mod-103-trajectory-and-tool-eval`](../mod-103-trajectory-and-tool-eval).
- LLM-as-judge scores attaching back to spans via the `app.judge_*` extensions → [`mod-104-llm-as-judge-in-product`](../mod-104-llm-as-judge-in-product).
- RAG-triad computed off the RETRIEVER span content and the LLM reply span content → [`mod-105-rag-eval-app-layer`](../mod-105-rag-eval-app-layer).
- CI enforcement of the schema MUST columns and regression fixtures produced by the bisection workflow → [`mod-106-eval-gated-cicd`](../mod-106-eval-gated-cicd).
- Sampled online-eval loop reading the schema you fixed here and using the sampling policy from chapter 06 → [`mod-107-online-eval-and-regression`](../mod-107-online-eval-and-regression).
- Guardrail-effectiveness measurement off `app.guardrail_verdict` and the GUARDRAIL span shape → [`mod-108-app-safety-and-guardrails-eval`](../mod-108-app-safety-and-guardrails-eval).
- Human review workflows fed by the trace annotations, labels, and reproducer fixtures from chapter 05 → [`mod-109-human-review-workflows`](../mod-109-human-review-workflows).
- Eval-data platform whose table definitions must agree with the `schema.version` you emit here → [`mod-110-eval-data-platform-slice`](../mod-110-eval-data-platform-slice).
- Cost / latency / quality dashboards rolling up from `app.cost_usd`, `app.ttft_ms`, `app.tpot_ms`, and span durations → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- Program-owner posture that treats the sampling + PII policy as part of the release-gate architecture and defers sign-off to the Governance-family peer → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
