# Wiring Traces Through Phoenix, Langfuse, Weave, Braintrust, and LangSmith

## Motivation

Once your app emits spans that follow one of the conventions from chapter 02, the next decision is where to send them. That decision has more downstream consequences than most: the backend you pick shapes what your on-call sees in the trace viewer (chapter 05), which sampling primitives are available (chapter 06), how easily mod-107's online eval loop can attach judge scores back to traces, and whether traces stay inside your VPC.

Five backends dominate the LLM-app observability space today: **Arize Phoenix**, **Langfuse**, **W&B Weave**, **Braintrust**, and **LangSmith**. Each is a real product with real docs; each takes a different position on conventions, transport, and hosting. This chapter walks the differences and hands you a decision framework — including a portable OTel-Collector recipe that lets you fan out to more than one, which is what most teams end up doing during a bake-off.

## Core concepts

### Two transport paths: SDK-direct vs OTLP

There are two ways for a trace to reach a backend:

1. **SDK-direct**: the vendor's SDK owns the span abstraction, records inputs / outputs, and posts to the vendor's ingest endpoint on its own schedule. Sometimes this SDK also emits OpenTelemetry spans under the hood; sometimes not.
2. **OTLP** (the OpenTelemetry Protocol, over gRPC or HTTP): your app uses the OpenTelemetry SDK, records spans in the standard shape, and exports via OTLP to either the backend directly (if it accepts OTLP) or to an **OpenTelemetry Collector** which fans out.

The OTLP + Collector path (transport documented at <https://opentelemetry.io/docs/specs/otlp/>, Collector at <https://opentelemetry.io/docs/collector/>) is the escape hatch that keeps you portable. You instrument once against OTel; the Collector runs a pipeline of receivers → processors → exporters; and you can add or remove backends by changing config, not code. Chapter 06 uses the same Collector to run tail sampling and PII scrubbing.

### The five backends at a glance

Each of these is a real product; consult the linked docs for feature-level detail rather than trusting a matrix. All names are trademarks of their respective owners.

**Arize Phoenix** — <https://docs.arize.com/phoenix>. Emitted-native for **OpenInference**. Available as a self-hosted container (`arize-phoenix`) or as part of Arize's hosted product. Ingests OTLP directly. Arize also maintains a family of auto-instrumentors named `openinference-instrumentation-*` for OpenAI, Anthropic, LangChain, LlamaIndex, DSPy, Bedrock, MistralAI, Groq, Vertex, and others — see <https://github.com/Arize-ai/openinference>. If the debugger workflow (chapter 05) is your primary use case, and you want a self-hostable viewer, Phoenix is a low-friction default.

**Langfuse** — <https://langfuse.com/docs>. Provides its own SDK (`langfuse` for Python, `langfuse-js` for TS) that models traces, sessions, and users as first-class objects. Also accepts OpenTelemetry OTLP (see the current docs URL for the ingest endpoint and configuration). Available as SaaS and as a self-hosted deployment (docs cover the self-host story explicitly). Session and user modelling are first-class in the UI — useful for chat / assistant surfaces where the *conversation*, not the individual span, is the debugging unit.

**W&B Weave** — <https://weave-docs.wandb.ai/>. SDK-first: instrumenting a function typically means adding a `@weave.op` decorator, and Weave records inputs, outputs, and metadata. Sits inside the Weights & Biases ecosystem — traces live next to experiment runs, datasets, and model evals. Weave supports OpenTelemetry OTLP ingestion — check the current docs for the endpoint. Strong fit if your team already lives in W&B.

**Braintrust** — <https://www.braintrust.dev/docs>. Positions itself around the **eval + tracing** loop rather than tracing alone; the primary artefact in Braintrust is an *experiment* comprised of scored traces, and the SDK is designed for that shape. Also supports OTel ingestion.

**LangSmith** — <https://docs.smith.langchain.com/>. LangChain's observability product. The `langsmith` SDK records traces from LangChain / LangGraph runtimes with essentially zero extra code, and LangSmith accepts OpenTelemetry OTLP as a general ingest path (see the docs). If your app is already LangChain-native, LangSmith is a cheap default because the runtime is instrumented for you.

<!-- needs-research: confirm current self-host offering for each of Weave, Braintrust, LangSmith on the next research cycle; hosting posture changes. -->

### A trade-off matrix

The columns below are the axes you actually decide on. The cells are directional, not scored — verify against current docs before you commit.

| Backend | Convention leaning | Self-host? | OTLP-native? | Session model | Agent / trajectory viewing | Eval integration | PII / hosting story |
|---|---|---|---|---|---|---|---|
| Phoenix | OpenInference | Yes | Yes | Present | Rich for OpenInference-shaped agent trees | Yes — evals attach to spans | Self-host inside your VPC is straightforward |
| Langfuse | Own model + OTel | Yes | Yes | First-class (sessions, users) | Good for chain / agent traces | Yes — scores attach to traces | Self-host + SaaS; check GDPR posture on the vendor docs |
| Weave | Own model + OTel | See vendor docs | Yes | Traces sit under W&B projects | Good | Yes — evals inside W&B | Primarily SaaS; see docs |
| Braintrust | Own model + OTel | See vendor docs | Yes | Traces belong to experiments | Good | Eval-first — this is the primary artefact | Primarily SaaS; see docs |
| LangSmith | Own model + OTel | See vendor docs | Yes | LangChain-native session model | Good — deep for LangChain / LangGraph | Yes — dataset + eval integration | Primarily SaaS; see docs |

Missing from this table on purpose: latency, cost, ingest throughput. Those are backend- and plan-specific and change frequently; do not hard-code them.

### The portable OTel-Collector recipe

If you want portability during the bake-off — or if you want to keep production shipping to one backend while an experiment ships to another — instrument against OTel once and fan out through the Collector. Sketch (real syntax; validate before deploying):

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:
  # See chapter 06 for tail_sampling and attribute-processor configs.

exporters:
  otlphttp/phoenix:
    endpoint: "https://<your-phoenix-host>/v1/traces"
    headers:
      # authentication per Phoenix docs
  otlphttp/langfuse:
    endpoint: "https://<region>.cloud.langfuse.com/api/public/otel/v1/traces"
    headers:
      # authentication per Langfuse docs

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/phoenix, otlphttp/langfuse]
```

The exact endpoint URLs, header names, and auth mechanics are backend-specific — cross-check each vendor's OpenTelemetry ingest page rather than copy-pasting a friend's config. The point is that a single instrumentation of your app supports N backends by adding exporters.

The Collector project also ships auto-generated exporters for other observability systems (Jaeger, Grafana Tempo, Datadog, etc.). If your team already runs Tempo or a Datadog trace ingestion path, the same Collector fans out there in parallel.

### Common instrumentation entry points

Regardless of backend, most teams end up with a stack like:

- **Framework auto-instrumentors** — `openinference-instrumentation-<framework>` for OpenAI, Anthropic, LangChain, LlamaIndex, Bedrock, MistralAI, Groq, Vertex, DSPy, and more (Arize's repo). Vendor-native SDKs (Langfuse, Weave, LangSmith) also auto-instrument LangChain / LlamaIndex / OpenAI depending on the vendor.
- **A layer of custom `app.*` attributes** you set on your own agent / chain / retriever boundaries (chapter 04).
- **W3C Trace Context propagation** across every network hop (chapter 01) so that downstream services (a tool API, a vector DB, an eval judge) join the same trace.

### Backend-driven trade-offs to watch

Some things change your architecture more than others:

- **Session as first-class** vs "sessions are a query on traces." If your surface is a chat, first-class sessions make debug work far shorter; grep the vendor's docs for the session model before you commit.
- **Eval-first data model** vs "traces are first, evals attach." Braintrust leans on the former; Phoenix and Langfuse lean on the latter. In practice you can do either work in any of them, but the primary artefact of the tool shapes what the on-call opens by default.
- **Auto-instrumentors' convention output.** Some auto-instrumentors emit OTel GenAI, some OpenInference, some vendor-native. The trace-viewer renders best on its native shape. If you mix auto-instrumentors from different sources, your viewer will render some spans richly and others as raw key-value.
- **Self-host feasibility.** For compliance surfaces (GDPR EU, healthcare, government), a self-hostable backend is often mandatory. Confirm the current self-host story per vendor — it moves.

### Decision guide

Rules of thumb — apply them per surface, not per company:

- *"I already use LangChain / LangGraph everywhere and don't want more code changes."* Start with **LangSmith** as the low-effort default; consider dual-shipping via OTel to a second backend during the bake-off.
- *"I need on-prem / self-hosted for compliance."* **Phoenix (self-host)** and **Langfuse (self-host)** are the two purpose-built LLM-app options; a Collector piping to Jaeger or Grafana Tempo is the vendor-neutral fallback. Confirm current self-host support for the others on their docs before ruling them out.
- *"My primary artefact is evaluations, not debugging."* **Braintrust** is designed around scored experiments; the trace view sits under an eval.
- *"Sessions dominate my debug work (support chat, meeting assistant)."* **Langfuse**'s first-class session model shortens the top-down trace walk.
- *"I want a lightweight debugger that reads OpenInference natively."* **Phoenix** is the most direct fit; the auto-instrumentor family is broad.
- *"I already live in the Weights & Biases universe."* **Weave** puts traces next to your runs.

For any of these choices, cross-shipping via the OTel Collector to a second backend for a two-to-four-week bake-off is cheap. Do the bake-off before committing.

### What all of these share

Independent of vendor, every viable backend for this track lets you:

- Ingest OTel-shaped traces (either natively OTLP or via a vendor SDK that emits standard traces).
- Render a nested trace tree with per-span latency and status.
- Show attributes and events on each span (message content, tool arguments, retrieved documents).
- Attach a **score** or **annotation** to a span or trace — the hook mod-104 uses for LLM-as-judge, mod-107 uses for sampled online eval, and mod-109 uses for human review.
- Filter and search across traces by attribute (session id, user, model version, experiment arm).

If a backend cannot do all five, it is not fit for the track's downstream modules. Rule it out.

## Example: a modest bake-off setup for a support-agent surface

Assume you have the surface from chapter 01: `agent.support_agent` with LLM planning, retrieval, tool call, LLM reply, and guardrail. You are choosing between Phoenix and Langfuse. Two-week bake-off:

1. **Instrument once, with OpenInference** using `openinference-instrumentation-openai` on the OpenAI client and manual OpenInference attributes on your agent / retriever / tool spans (chapter 04).
2. **Deploy an OTel Collector sidecar** with two exporters — one pointed at a self-hosted Phoenix, one pointed at Langfuse (SaaS or self-host).
3. **Add mirror attributes** where auto-instrumentors emit OTel GenAI (`gen_ai.*`) so that both viewers render the model call the same way (chapter 02).
4. **Run 200 real user sessions through each viewer.** On-call opens each viewer during a synthetic bad-run drill (chapter 05). Score:
   - Time to open the failing trace from a bug report.
   - Time to identify the root-cause span.
   - Legibility of retrieved-document rendering.
   - Ease of attaching a judge score for a hypothetical mod-107 online-eval loop.
5. **Decide on the primary**, keep the other as a mirror for four weeks, then drop it. Note in the mod-101 eval plan which backend is primary and which is deferred.

If your scorecard cannot separate them on your workload, either backend is fine — pick the one with the easier compliance story for your team.

## Summary

- Two transports: **SDK-direct** (vendor SDK owns the span) and **OTLP** (OTel SDK plus an OpenTelemetry Collector). OTLP + Collector is the portable path.
- Five common backends today: **Phoenix**, **Langfuse**, **Weave**, **Braintrust**, **LangSmith**. Each accepts OTLP; each has its own SDK; each has its own strengths.
- The trade-off matrix that matters: convention leaning, self-host feasibility, session model, eval integration, agent-trace viewing quality, PII / hosting posture. Verify against current docs; do not commit to a matrix in your head.
- Instrument once against OTel, then fan out via the Collector during any bake-off. Commit only after two weeks of production-shaped traffic.
- Do NOT invent latency, cost, or throughput numbers to justify a choice — read the vendor docs and re-check on release.

Chapter 04 designs the span-attribute schema this backend has to render.
