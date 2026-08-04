# exercise-01: OTel GenAI and OpenInference Semantic Conventions

**Estimated effort:** 2 hours

## Objective

Instrument the same small "chat plus one tool" service twice — once emitting **OpenTelemetry GenAI** conventions, once emitting **OpenInference** conventions — and produce a short written comparison of the two shapes on a real trace viewer. The point is not to *pick* a winner today; the point is to build the muscle of reading a spec, laying down attribute names deliberately, and noticing where the two conventions collide, mirror, or disagree.

By the end you will have hands-on evidence for chapter 02's "pick a primary per span, mirror if you must" rule — evidence you will lean on in exercise-03 when you design the schema for real.

## Prerequisites

- Chapters 01 and 02 of this module.
- Python 3.11+ (or the language your team uses — the exercise translates cleanly to TS / Java / Go, but the sample snippets use Python).
- An OpenAI, Anthropic, Bedrock, or equivalent model API key with a small budget cap on it (you will send ~30 requests total).
- A local trace viewer that will render both conventions well. **Arize Phoenix** run in Docker (`docker run -p 6006:6006 -p 4317:4317 arizephoenix/phoenix:latest` — check the current tag on <https://docs.arize.com/phoenix>) is the simplest fit because it speaks both. Either Langfuse or the vanilla Jaeger UI works too if you already have one; note that a vanilla viewer will render OpenInference less richly than Phoenix does, which is itself part of the lesson.
- The OpenTelemetry Python SDK (`opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp`).

## Set-up

1. Write a minimal "support Q&A" service — one Python function `answer(question: str) -> str` — that:
   - Calls the model with a short system prompt.
   - If the model emits a tool call to a stubbed `lookup_order(order_id: str)` tool, executes the stub (return a JSON dict from an in-memory table of five orders).
   - Calls the model a second time with the tool result and returns the final assistant text.
2. Wire the OTel SDK once, pointing at your local Phoenix / Langfuse / Jaeger instance over OTLP-HTTP.
3. Prepare a small deterministic set of five inputs. Two should trigger the tool call (`"Where is order 42?"`, `"What's the ETA on 87?"`); three should not (`"Do you ship to Iceland?"`, greeting, off-topic).

Do NOT use an auto-instrumentor for this exercise. The whole point is to write the attribute names yourself so you feel the shape of each convention.

## Requirements

Produce a single directory `mod-102/exercise-01/` in your working repo containing:

1. **`app_otel_genai.py`** — the service instrumented with **OpenTelemetry GenAI** attributes only. Every LLM span carries:
   - `gen_ai.system`, `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.request.temperature`, `gen_ai.request.max_tokens`.
   - `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons`, `gen_ai.response.id`.
   - `gen_ai.tool.name` and `gen_ai.tool.call.id` on the tool-related spans.
   - Message content on span **events** (`gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.system.message`, `gen_ai.choice`) — carry content on events for this variant even if the current spec has evolved to also allow attributes.
   - `SpanKind.CLIENT` on the model call, `SpanKind.INTERNAL` on the tool execution.
2. **`app_openinference.py`** — the same service instrumented with **OpenInference** attributes only. Every LLM span carries:
   - `openinference.span.kind=LLM`, `llm.provider`, `llm.model_name`, `llm.invocation_parameters`.
   - `llm.input_messages.<i>.message.role/content` and `llm.output_messages.<i>.message.role/content` on indexed sub-attributes (content-on-attributes).
   - `llm.token_count.prompt/completion/total`.
   - `llm.tool_calls.<i>.tool_call.function.name/arguments` where relevant.
   - `input.value` / `output.value` for the top-of-tree convenience string.
   - A `TOOL`-kind span for the stubbed tool with `tool.name`, `tool.description`, `input.value`, `output.value`.
3. **`compare.md`** — 400–600 words in three sections:
   - **Attribute-name diff table.** One row per concept (system / provider, requested model, served model, prompt tokens, completion tokens, finish reason, tool name, tool arguments, message content, tool return). Columns: `Concept`, `OTel GenAI name`, `OpenInference name`, `Notes` (mirror, only-one-side, shape difference).
   - **Rendering diff.** For each of the five inputs, one line naming what the viewer showed by default under each variant. Concrete: "under OpenInference, message list rendered as chat bubbles with roles; under OTel GenAI, message content appeared under a 'span events' tab as `gen_ai.user.message`."
   - **Which one you would keep, why, and where you would mirror.** Two paragraphs. If you would keep neither on its own — a legitimate answer — describe the dual-annotation you would ship instead, using the pattern in chapter 02.

## Starter guidance

- Copy the two snippets in chapter 02 as your starting scaffolds. They intentionally omit tool-call and streaming details; you will fill them in.
- Do not paper over provider-side differences. If the OpenAI Python SDK's response object exposes `response.model` and yours does not, note it in `compare.md` — that friction *is* part of the deliverable.
- Do not turn on any auto-instrumentation. The value of this exercise is fully manual instrumentation of ~50 lines of code. You will run auto-instrumentors in the lab.
- If your viewer is Phoenix, open the trace under both variants side by side. Screenshot each is fine but not required; the written comparison is what mod-107 reviewers will read.
- Budget-guard the run. Five inputs × two variants × two model calls each ≈ 20 model calls; keep temperature at 0.2 and `max_tokens` low.

## Acceptance criteria

You are done when:

- Both `app_otel_genai.py` and `app_openinference.py` run end-to-end on the same five inputs and produce traces that appear in your viewer.
- Each variant's traces carry every attribute in the requirement list — grep for the attribute names to confirm.
- Content is on **events** for the OTel GenAI variant and on **indexed attributes** for the OpenInference variant. The two shapes are visibly different in the viewer.
- The tool call and its return appear as a distinct child span of the second model call under both variants (naming differs; the causal structure is the same).
- `compare.md` contains the diff table, the rendering diff for all five inputs, and your keep-and-mirror recommendation.
- Nothing in the repo hard-codes a secret; the model API key is read from an environment variable.

## Stretch goals

- Add a third variant `app_dual.py` that emits **both** conventions on the same LLM span, using the mirror pattern from chapter 02. Verify the viewer still renders correctly and that no field disagrees between the two conventions.
- Enable streaming on one model call and add `app.ttft_ms` and `app.tpot_ms` attributes; note in `compare.md` how each convention (or neither) handles first-token / per-token latency.
- Add a second child model call inside the tool execution (e.g., a summarisation of the retrieved order) so you have a *nested* LLM span. Confirm the trace tree remains legible.

## What this exercise does *not* cover

You are not designing the schema yet — that is exercise-03. You are not deciding the backend — that is exercise-02. You are not sampling or scrubbing — that is exercise-05. Stay narrow: two conventions, five inputs, one comparison.
