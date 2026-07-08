# OpenTelemetry GenAI and OpenInference Semantic Conventions

## Motivation

Chapter 01 defined *what* a trace is. This chapter fixes *what to call the fields on it*. If your team names the model version `model`, another team names it `llm.model`, and a third names it `gen_ai.request.model`, no trace viewer, no eval loop, and no cross-service correlation works. Semantic conventions are agreements on the field names. They are also fragile: two different agreements are active in the LLM space right now, and picking one — or knowing how to translate — is a real design decision.

The two live conventions this module cares about are the **OpenTelemetry GenAI semantic conventions** (an OpenTelemetry-hosted spec, currently in development / experimental status) and **OpenInference** (an Arize-hosted spec, richer per-span-kind attribute set targeted at LLM-app debugging). They overlap significantly, they diverge in a few important places, and the backends in chapter 03 lean differently on the two. This chapter is where you learn the vocabulary of each, and the practical rule that keeps you out of trouble: pick one convention as primary per span, and translate if you need to.

## Core concepts

### What a semantic convention is

A **semantic convention** is a document that says: *for this kind of span, use these attribute names, with these types, with these values*. It is not an SDK. It is a schema contract. Backends that understand the convention can render the span meaningfully — a "model" field, a "prompt" panel, a "token cost" chip, a tool-call widget. Backends that do not understand it fall back to a generic key-value view. Two teams that emit the same convention can dual-ship their traces to the same viewer without translation.

The concrete file for OpenTelemetry's semantic conventions repo lives at <https://github.com/open-telemetry/semantic-conventions>; the human-readable rendering is at <https://opentelemetry.io/docs/specs/semconv/>. The GenAI section is under <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. OpenInference lives at <https://github.com/Arize-ai/openinference> — the semantic conventions file is under `spec/`.

### OpenTelemetry GenAI semantic conventions

Status: currently in **development / experimental**. The attribute names have evolved and will keep evolving until they graduate to stable. Do not pin your code to a version of the spec in your head — link the URL, and rely on the OpenTelemetry SDK's `semconv` package once the version you need is published.

Core attributes you can expect on an LLM span (the exact set may shift; check the spec):

| Attribute | Type | Example | Notes |
|---|---|---|---|
| `gen_ai.system` | string | `openai`, `anthropic`, `aws.bedrock`, `azure.ai.openai`, `cohere`, `ibm.watsonx.ai` | Provider / system identifier |
| `gen_ai.operation.name` | string | `chat`, `text_completion`, `embeddings`, `generate_content`, `execute_tool`, `invoke_agent` | The operation the span records |
| `gen_ai.request.model` | string | `gpt-4o-2024-08-06` | The model *requested* — may differ from the served version |
| `gen_ai.response.model` | string | `gpt-4o-2024-08-06` | The actual served version (from the response body) |
| `gen_ai.request.temperature` | float | `0.2` | Decoding parameter |
| `gen_ai.request.top_p` | float | `0.95` | Decoding parameter |
| `gen_ai.request.max_tokens` | int | `1024` | Cap on generation |
| `gen_ai.request.stop_sequences` | string[] | `["\n\n"]` | Cap on generation |
| `gen_ai.usage.input_tokens` | int | `842` | Prompt token count |
| `gen_ai.usage.output_tokens` | int | `117` | Completion token count |
| `gen_ai.response.finish_reasons` | string[] | `["stop"]`, `["tool_calls"]`, `["length"]` | Why the generation ended |
| `gen_ai.response.id` | string | `chatcmpl-…` | Provider-side call id |
| `gen_ai.tool.name` | string | `open_ticket` | On tool-call spans |
| `gen_ai.tool.call.id` | string | `call_abc123` | Matches the `id` on the model's tool-call payload |

Note the **operation.name** vocabulary — the spec covers a set of operations beyond `chat`, including `execute_tool` and `invoke_agent`. That matters at application altitude because an agent's planning-and-tool loop can be recorded as `invoke_agent` at the root and `chat` / `execute_tool` on its children, which is exactly the shape a trace viewer wants.

**Content — attributes vs events.** Historically, the OpenTelemetry GenAI spec used *span events* to carry message content: an event named `gen_ai.user.message` for the user turn, `gen_ai.assistant.message` for the assistant turn, `gen_ai.system.message` for the system prompt, and `gen_ai.choice` for each returned choice. Content-on-events keeps the span's attribute set small and lets a backend that does not want to store payloads discard events. The spec has been evolving toward carrying inputs and outputs as attributes as well (see the URL above for the current shape). The practical guidance: pick one shape per surface, apply it consistently, and re-read the spec on the URL before you cut a release — this is one of the surfaces that changes.

<!-- needs-research: track the current OTel GenAI spec status on message content (attribute vs event) each release cycle. -->

### OpenInference semantic conventions

OpenInference is Arize's convention, deliberately richer at the LLM-app layer and organised around a per-span-kind vocabulary. Repo: <https://github.com/Arize-ai/openinference>.

The central attribute is `openinference.span.kind`, which classifies the span. Documented values *include* (this list is meant to be indicative, not exhaustive — read the spec):

`LLM`, `CHAIN`, `RETRIEVER`, `TOOL`, `EMBEDDING`, `RERANKER`, `AGENT`, `GUARDRAIL`, `EVALUATOR`.

That single field is what lets a trace viewer render an LLM span differently from a retrieval span differently from a tool span, even when the underlying framework is homegrown.

Attributes that apply across all kinds:

| Attribute | Type | Notes |
|---|---|---|
| `input.value` | string | The raw input (prompt, question, tool arguments, retrieved query, …) |
| `input.mime_type` | string | `text/plain`, `application/json`, … |
| `output.value` | string | The raw output |
| `output.mime_type` | string | Type of the output |
| `metadata` | JSON | Freeform per-span metadata |
| `tag.tags` | string[] | Free-tag search |
| `session.id`, `user.id` | string | Session / user identity (see chapter 04 for hashing) |

Per-kind attributes that matter for this module:

**LLM span**

| Attribute | Notes |
|---|---|
| `llm.model_name` | The model — parallel to `gen_ai.response.model` |
| `llm.provider` | The provider — parallel to `gen_ai.system` |
| `llm.system` | Provider system id (backend usage varies) |
| `llm.input_messages.<i>.message.role` | `system`, `user`, `assistant`, `tool` |
| `llm.input_messages.<i>.message.content` | Message content |
| `llm.output_messages.<i>.message.role` | Reply role |
| `llm.output_messages.<i>.message.content` | Reply content |
| `llm.token_count.prompt`, `llm.token_count.completion`, `llm.token_count.total` | Token usage |
| `llm.invocation_parameters` | JSON blob for decoding parameters and provider-specific extras |
| `llm.prompt_template.template` | The template string (or a stable hash — see chapter 04) |
| `llm.prompt_template.variables` | JSON of the variable values used |
| `llm.tools` | JSON of the tool schemas passed to the model |
| `llm.tool_calls.<i>.tool_call.function.name` | Tool the model wants to call |
| `llm.tool_calls.<i>.tool_call.function.arguments` | Arguments the model produced (JSON string) |

**Tool span**

| Attribute | Notes |
|---|---|
| `tool.name`, `tool.description` | Tool identity |
| `tool.parameters` | Tool's declared parameter schema |
| `input.value` | Parsed arguments the model produced |
| `output.value` | Tool's return |

**Retriever span**

| Attribute | Notes |
|---|---|
| `retrieval.documents.<i>.document.id` | Stable id in your doc store |
| `retrieval.documents.<i>.document.content` | Chunk content (respect content-size caps — chapter 04) |
| `retrieval.documents.<i>.document.score` | Retrieval score (higher-is-better convention) |
| `retrieval.documents.<i>.document.metadata` | JSON of chunk metadata |
| `input.value` | The retrieval query |

The **indexed sub-attribute** shape (`llm.input_messages.<i>.message.content`) is how OpenInference records lists on flat key-value spans without a schema for nested types. Backends that speak OpenInference natively (chapter 03) render these as message lists in the UI. Backends that do not still see them as ordinary attributes.

### How the two conventions relate today

Both conventions are active. They overlap on the identity attributes (system, model, tokens) and diverge on:

- **Content shape.** OTel GenAI historically used events for content; OpenInference uses indexed attributes on the span itself.
- **Span-kind vocabulary.** OTel GenAI uses `SpanKind` + `gen_ai.operation.name`; OpenInference adds `openinference.span.kind` with a richer LLM-app vocabulary.
- **Retrieval / tool granularity.** OpenInference has an explicit RETRIEVER kind with per-document sub-attributes. OTel GenAI has been evolving in this area — check the spec.
- **Agent operations.** OTel GenAI has `invoke_agent` / `execute_tool` operations; OpenInference has `AGENT` / `TOOL` kinds. These are compatible — you can set both.

Practical framings:

| If you use … | You will most likely emit … |
|---|---|
| Provider SDKs that are OTel-native (increasingly common for OpenAI, Anthropic, Bedrock) | OTel GenAI conventions |
| Arize's `openinference-instrumentation-*` auto-instrumentors for popular frameworks (LangChain, LlamaIndex, DSPy, OpenAI, Anthropic, Vertex, Bedrock, MistralAI, Groq, …) | OpenInference conventions |
| A homegrown chain / agent runtime | Whichever convention you pick, applied consistently, plus custom `app.*` extensions where the convention is silent |

**Never** emit half of one and half of the other on the same span — it makes a trace viewer's rendering unreliable. If you need both (because two auto-instrumentors write different names), pick a primary and set the other as a *mirror* attribute explicitly.

### A minimal manual OTel snippet

Instrumenting a single chat call manually, in Python, with the OTel SDK and OTel GenAI attributes:

```python
from opentelemetry import trace
from opentelemetry.trace import SpanKind

tracer = trace.get_tracer("acme.support_agent")

with tracer.start_as_current_span(
    "llm.plan",
    kind=SpanKind.CLIENT,
    attributes={
        "gen_ai.system": "openai",
        "gen_ai.operation.name": "chat",
        "gen_ai.request.model": "gpt-4o-2024-08-06",
        "gen_ai.request.temperature": 0.2,
        "gen_ai.request.max_tokens": 512,
    },
) as span:
    resp = client.chat.completions.create(...)

    span.set_attribute("gen_ai.response.model", resp.model)
    span.set_attribute("gen_ai.response.id", resp.id)
    span.set_attribute("gen_ai.usage.input_tokens", resp.usage.prompt_tokens)
    span.set_attribute("gen_ai.usage.output_tokens", resp.usage.completion_tokens)
    span.set_attribute(
        "gen_ai.response.finish_reasons",
        [c.finish_reason for c in resp.choices],
    )

    # Content on events (historical GenAI shape).
    span.add_event("gen_ai.user.message", attributes={"content": user_msg})
    span.add_event(
        "gen_ai.assistant.message",
        attributes={"content": resp.choices[0].message.content or ""},
    )
```

The equivalent instrumentation using OpenInference (content-on-attributes, span-kind vocabulary) trades events for indexed attributes:

```python
with tracer.start_as_current_span("llm.plan", kind=SpanKind.CLIENT) as span:
    span.set_attributes({
        "openinference.span.kind": "LLM",
        "llm.provider": "openai",
        "llm.model_name": "gpt-4o-2024-08-06",
        "llm.invocation_parameters": json.dumps({
            "temperature": 0.2,
            "max_tokens": 512,
        }),
        "llm.input_messages.0.message.role": "system",
        "llm.input_messages.0.message.content": SYSTEM_PROMPT,
        "llm.input_messages.1.message.role": "user",
        "llm.input_messages.1.message.content": user_msg,
        "input.value": user_msg,
        "input.mime_type": "text/plain",
    })

    resp = client.chat.completions.create(...)

    span.set_attributes({
        "llm.model_name": resp.model,
        "llm.token_count.prompt": resp.usage.prompt_tokens,
        "llm.token_count.completion": resp.usage.completion_tokens,
        "llm.token_count.total": resp.usage.total_tokens,
        "llm.output_messages.0.message.role": "assistant",
        "llm.output_messages.0.message.content": resp.choices[0].message.content or "",
        "output.value": resp.choices[0].message.content or "",
        "output.mime_type": "text/plain",
    })
```

The two snippets record the same call and both will land in most backends. Which shape you prefer depends on which backend renders it best (chapter 03), how comfortable you are with events vs indexed attributes on your backend's ingest path, and whether your auto-instrumentors already emit one shape or the other.

Neither snippet is what you would run in production — production uses auto-instrumentation (`openinference-instrumentation-openai`, or the provider SDK's built-in OTel hooks) and layers your custom `app.*` and `session.*` / `user.*` attributes on top. Chapter 04 designs that top layer.

### Custom extensions

Both conventions leave space for domain attributes. Namespace them so a reader can tell what is standard and what is yours:

- `app.surface` — the surface name (see mod-101).
- `app.step` — a stable name for the pipeline step (`plan`, `retrieve`, `tool.open_ticket`, `reply`).
- `experiment.arm` — the A/B or canary arm.
- `feature_flag.<name>` — feature-flag values active on this request.

Do NOT re-use a `gen_ai.*` or `llm.*` name for a custom purpose. That collides with future spec evolution.

## Example: dual-convention on the same LLM span

Suppose you cannot pick — your homegrown code emits OpenInference (because your viewer is Phoenix) but your auto-instrumented OpenAI SDK writes OTel GenAI. On the LLM span you own, dual-annotate deliberately:

```python
span.set_attributes({
    # Primary: OpenInference (viewer-native).
    "openinference.span.kind": "LLM",
    "llm.model_name": "gpt-4o-2024-08-06",
    "llm.token_count.prompt": 842,
    "llm.token_count.completion": 117,
    # Mirror: OTel GenAI (interop).
    "gen_ai.system": "openai",
    "gen_ai.request.model": "gpt-4o-2024-08-06",
    "gen_ai.response.model": "gpt-4o-2024-08-06",
    "gen_ai.usage.input_tokens": 842,
    "gen_ai.usage.output_tokens": 117,
})
```

Both names hold the same value. Store the primary once in your schema doc (chapter 04) so downstream mod-107 / mod-110 code always reads the same field.

## Summary

- Semantic conventions are field-name contracts, not SDKs. Pick one as **primary per span** and stay consistent.
- **OpenTelemetry GenAI** is the OTel-hosted convention; currently in development / experimental. Core attributes are `gen_ai.system`, `gen_ai.operation.name`, `gen_ai.request.*`, `gen_ai.response.*`, `gen_ai.usage.*`. Content historically travels on span events (`gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.choice`).
- **OpenInference** is Arize's convention with a richer per-span-kind vocabulary via `openinference.span.kind` (LLM / TOOL / RETRIEVER / CHAIN / AGENT / EMBEDDING / RERANKER / GUARDRAIL / EVALUATOR, among others) and indexed attributes for messages, tool calls, and retrieved documents.
- The two overlap on identity and diverge on content shape and per-kind granularity. Many backends accept either.
- Namespace custom extensions with `app.*`, `experiment.*`, `feature_flag.*` — do not overload spec-owned names.
- Re-read the URL each release cycle; both specs move.

Chapter 03 walks the backends that consume these conventions and how to wire OTLP export to them.
