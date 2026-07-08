# Why Trace-Level Instrumentation for LLM Apps

## Motivation

Application-altitude evaluation only works if you can answer a very specific question about a very specific past run: *what did that call chain actually do*. Not "the p95 latency last hour"; not "the average faithfulness score today"; not "how many tool calls did we serve." Those numbers are downstream. The upstream artefact is the **call tree** of a single user's interaction — the prompt that entered the model, the retrieved chunks that grounded it, the tool call the model emitted, the tool's return, the second model call that composed the reply, the guardrail that ran on the way out, the cost of each step. Without that tree, mod-101's SLIs are unfalsifiable: you can print a number, but you cannot bisect it back to the change that moved it.

That call tree is a **trace**. Recording it well is what this module teaches. The rest of the track — trajectory eval (mod-103), judge scoring (mod-104), RAG-triad (mod-105), CI gates (mod-106), the online-eval loop (mod-107), guardrail measurement (mod-108), human review (mod-109), the data platform (mod-110), cost/latency trade-offs (mod-111) — every one of them consumes traces of the exact shape you decide here. If you skip this module and hand mod-107 a stream of `print`-lines instead of spans, mod-107 does not work: it can neither sample intelligently, nor route a bad run to the review queue, nor tell whether a regression is on the retriever or the reply LLM.

This first chapter fixes the vocabulary the rest of the module reuses — *trace*, *span*, *span event*, *span kind*, *span link*, *context propagation* — and states the five debugger questions a trace has to answer for this track's downstream work to succeed.

## Core concepts

### Trace, span, event

The vocabulary is inherited from distributed tracing, and it is worth reusing verbatim rather than reinventing:

- A **trace** is one logical operation, end-to-end, identified by a 128-bit `trace_id`. For an LLM app, a trace is typically one user interaction (one message in a chat, one API call to your service).
- A **span** is one unit of work inside the trace: a model call, a tool call, a retrieval, an agent's planning step, a guardrail check. Each span has a 64-bit `span_id`, a start time, an end time, a status, and a set of attributes.
- Spans are **hierarchical**: every span except the root has a `parent_span_id`. The tree of spans is what a trace viewer renders.
- A **span event** is a timestamped record attached to a span — historically the OpenTelemetry GenAI conventions used span events to carry message content (`gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.choice`) rather than putting large prompts on attributes. Chapter 02 goes into this.
- A **span link** is a pointer from one span to another trace/span, used when the causal graph is not strictly a tree — e.g., an agent that fans out sub-agents into their own traces and later joins the results.

The **W3C Trace Context** spec (<https://www.w3.org/TR/trace-context/>) defines how `trace_id` and `span_id` propagate across process boundaries via HTTP headers (`traceparent`, `tracestate`). Any inter-service call in your app — the model API, the vector DB, the CRM tool — should propagate context so the whole chain shows up in one trace.

### Span kind

`SpanKind` (from the OpenTelemetry base spec — see <https://opentelemetry.io/docs/specs/otel/trace/api/#spankind>) partitions spans by role in a distributed system:

| Kind | Meaning | Typical LLM-app use |
|---|---|---|
| `INTERNAL` | In-process work | Chain / agent planning steps, guardrail checks, hashing |
| `CLIENT` | Outbound network request | Model API call, tool HTTP call, vector-DB query |
| `SERVER` | Inbound network request | Your API endpoint that received the user request |
| `PRODUCER` / `CONSUMER` | Async messaging | Job queue offload of long tool calls |

`SpanKind` is *orthogonal* to the LLM-specific vocabulary (OpenInference's `openinference.span.kind` with values LLM, TOOL, RETRIEVER, CHAIN, AGENT, ...). Chapter 02 explains how the two live together; you set both.

### Why a trace, not a log line

Once you have anything beyond a single flat model call, per-request logging stops describing the system. Consider the same failure as it looks in three telemetry shapes:

| Telemetry | What the on-call sees | What they can do about it |
|---|---|---|
| Log lines (one per stage) | `plan="..." tool_call="..." reply="..."` interleaved across N sessions | Grep by session id; hope you can reconstruct the order |
| Metrics only | `llm_call_total{model="gpt-4o"} +1` and `p95_latency 4.2s` | Confirm the surface is slow; cannot explain why |
| Trace (call tree) | The whole tree for the failing request, span by span, with parent/child, times, attributes, and events | Bisect (chapter 05); reproduce (chapter 05); route to review (mod-109) |

The trace is not "verbose logging." It is a *structured* record of causality. Structure is what makes an eval loop mechanised instead of manual.

### The five debugger questions

Every trace you emit in this module must let a debugger answer these five questions **on the trace alone**, without diving into git blame or reproducing the request in staging:

1. **What was the exact prompt / input to this call?** — content on the LLM span (attribute or event; see chapter 02).
2. **What did the model return, in full?** — content on the LLM span; including finish reason and any tool-call payloads.
3. **Which retrieved chunks / tools / sub-agents did it call, in what order?** — child spans of kind RETRIEVER / TOOL / AGENT, with their own inputs / outputs / status.
4. **How much did this call cost and how long did it take?** — token usage on the LLM spans (`gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`), latency from span duration, cost either as an attribute or computed from usage + a price table.
5. **Which user / session / feature-flag context did it happen under?** — root-span attributes: hashed user id, session id, surface name, `service.version`, experiment arm.

If your trace cannot answer one of these five, the trace-viewer workflow in chapter 05 breaks. Chapter 04 designs the schema; chapters 05 and 06 use it.

### How LLM traces differ from a normal microservice trace

A microservice trace is a tree of RPCs. An LLM trace is that, plus a set of things a normal APM was not built for:

- **The payload is the point.** A REST trace usually records that a call happened, not what it said. An LLM trace needs the actual prompt and completion text, which is (a) large, (b) sensitive, (c) sometimes streamed.
- **Streaming.** The response arrives token-by-token; time-to-first-token (TTFT) and time-per-output-token (TPOT) are first-class attributes distinct from wall-clock duration.
- **Tool-calls-as-outputs.** The model's *output* on one span is often the *input* to a tool span the runtime then executes. The trace tree encodes that hand-off.
- **Cost per span.** For a normal RPC nobody instruments dollars per call. For LLM calls, cost is a first-class SLI — the trace has to carry the tokens.
- **PII in the payload.** The user's raw message is often in `input.value`. Chapter 06 addresses scrubbing at the source and at the Collector.
- **Cardinality.** Metric labels usually stay bounded (endpoint, status). LLM attributes carry model versions, prompt-template hashes, tool names, experiment arms; the cardinality shape drives the choice between attribute vs event and, downstream, the sampling policy.

### A worked call tree

Consider a small support-agent surface — one agent that plans, retrieves, calls a `open_ticket` tool if needed, and drafts a reply. A well-shaped trace for that surface looks like this:

```
trace 4b8a5c…
└─ agent.support_agent                            [openinference.span.kind=AGENT]
   ├─ llm.plan                                   [kind=LLM, gen_ai.operation.name=chat]
   ├─ retriever.docs.search                      [kind=RETRIEVER]
   │  ├─ document 0  score=0.81  id=doc-142
   │  ├─ document 1  score=0.78  id=doc-91
   │  └─ document 2  score=0.55  id=doc-208
   ├─ tool.open_ticket                           [kind=TOOL, tool.name=open_ticket]
   ├─ llm.reply                                  [kind=LLM, gen_ai.operation.name=chat]
   └─ guardrail.output_policy                    [kind=GUARDRAIL]
```

Every arrow is a parent-child edge. `agent.support_agent` is the root; its duration is the end-to-end latency for the surface. The two LLM spans carry the actual prompts and completions. `retriever.docs.search` carries the ranked chunks (id + score + content, per chapter 04). `tool.open_ticket` carries the arguments the model produced and the tool's return. `guardrail.output_policy` records whether the guardrail rewrote or blocked the reply.

Given only this tree, you can answer the five debugger questions (chapter 05 walks a bad-run bisection against exactly this shape). Given only a log line saying "reply generated in 4.2s", you can answer none of them.

### What this module produces

By the end of the module, you should be able to:

- Instrument an existing LLM app end-to-end with OpenTelemetry GenAI or OpenInference conventions (chapter 02).
- Choose a backend and wire OTLP export to it, defensibly (chapter 03).
- Design the span-attribute schema that supports your surface's SLIs (chapter 04).
- Walk a trace top-down and pin a bad run's root cause (chapter 05).
- Sample intelligently under a cost budget and scrub PII before it leaves your VPC (chapter 06).

The concrete artefact from the exercises is a running instrumented app plus a written policy — the same artefact that mod-107 will drive a sampled-trace eval loop against.

## Summary

- A **trace** is the call tree of one user interaction; a **span** is one step inside it. Reuse the OpenTelemetry vocabulary rather than inventing your own.
- LLM traces differ from normal APM traces on five dimensions: payload-is-the-point, streaming, tool-calls-as-outputs, cost-per-span, PII-in-payload.
- Every trace must answer five debugger questions: exact prompt, exact response, tool / retrieval / sub-agent order, cost and latency, user / session / arm context.
- Propagate W3C Trace Context (`traceparent`, `tracestate`) across every network hop so the model call, the tool call, the vector DB, and the CRM show up in one trace.
- The output of this module is a running app whose traces let mod-103 through mod-112 do their jobs. The rest of the track assumes trace instrumentation of the shape this module builds.

Chapter 02 walks the two semantic conventions that give those spans their attribute names.
