# quiz-01: Trace-Level Instrumentation for LLM Apps and Agents

**Estimated time:** 15–20 minutes
**Covers:** chapters 01–06 of this module.

Answer each question, then check the key at the bottom. Explanations cite the chapter the answer comes from so you can re-read the relevant section if an answer surprised you.

---

## Questions

### 1. Chapter 01 lists five debugger questions a trace must answer *on the trace alone*. Which of the following is *not* one of them?

- A. What was the exact prompt / input to this call?
- B. What did the model return, in full?
- C. Which retrieved chunks / tools / sub-agents did it call, in what order?
- D. Which feature flag was flipped most recently across the fleet?

### 2. Which claim about how LLM traces differ from a normal APM trace is chapter 01's actual framing?

- A. The payload is the point — a REST trace records that a call happened, an LLM trace needs the actual prompt and completion text, which is large, sensitive, and sometimes streamed.
- B. LLM traces do not need W3C Trace Context propagation because every model call is in-process.
- C. Cost per call is not instrumented on LLM spans; it is computed later from billing exports.
- D. TTFT and TPOT are the same as wall-clock duration for streamed responses.

### 3. An auto-instrumentor emits `gen_ai.response.model` on an LLM span. Your homegrown wrapper wants to annotate the same span with `llm.model_name` for your Phoenix viewer. Chapter 02's practical rule says:

- A. Never put both; pick one and delete the other with a processor.
- B. Pick one convention as **primary per span** and, if you need both, deliberately mirror — holding the same value on both attributes and documenting the primary in the schema doc.
- C. Emit half the OTel GenAI fields and half the OpenInference fields on the same span so both backends can "see" something.
- D. Rename `gen_ai.response.model` to `llm.response.model` to unify the namespaces.

### 4. Chapter 02 describes two shapes for carrying message content on an LLM span. Which pair matches the two conventions as the chapter describes them?

- A. OTel GenAI uses indexed attributes on the span (`llm.input_messages.<i>.message.content`); OpenInference uses span events.
- B. OTel GenAI historically uses span events (`gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.system.message`, `gen_ai.choice`); OpenInference uses indexed attributes on the span itself.
- C. Both put content exclusively in a separate `messages` resource outside the span.
- D. Both forbid content on the span — content must always live in blob storage.

### 5. Chapter 03 names two transport paths for traces reaching a backend. Which description matches the chapter's framing?

- A. SDK-direct ships via vendor SDK to the vendor endpoint; OTLP is OpenTelemetry Protocol over gRPC or HTTP — SDK-direct is the portable escape hatch.
- B. SDK-direct ships via vendor SDK to the vendor endpoint; OTLP + an OpenTelemetry Collector is the portable path because you instrument once against OTel and change backends by editing config rather than code.
- C. Both transports require a vendor SDK; the difference is which cloud region ingests the traces.
- D. OTLP only works if you already run Jaeger; SDK-direct is required for any LLM-specific viewer.

### 6. A team must self-host their trace backend for a compliance surface and wants the viewer to render OpenInference natively. Which pair of backends does chapter 03 call out as *purpose-built LLM-app options that support self-host*?

- A. Phoenix and Langfuse.
- B. Braintrust and LangSmith.
- C. W&B Weave and LangSmith.
- D. Datadog and Braintrust.

### 7. Chapter 04 forbids putting the raw user id or email on a span attribute. What recipe does the chapter recommend for `user.id`, and what is its legal status under GDPR?

- A. SHA-256 of the raw id with no salt; anonymous under Art. 4(1).
- B. Plaintext user id with access controls at the backend; still PII but acceptable.
- C. HMAC-SHA256 of the raw id keyed with a per-tenant salt held in a KMS; **pseudonymous** under Art. 4(5), so personal-data obligations still apply.
- D. Random UUID generated per request, with no mapping back to the user.

### 8. Chapter 04 states a content-granularity rule to prevent the same 20 KB of text from appearing three times in a trace tree. Which phrasing matches?

- A. Full-fidelity content lives on every ancestor span so top-down walks always see it.
- B. Full-fidelity content lives on the LLM span it belongs to; the CHAIN / AGENT parent carries a redacted repro summary on `output.value`, not the raw model text.
- C. Content lives only on the root span and is referenced by child spans with a pointer.
- D. Content is forbidden on spans; put it in logs and correlate by `trace_id`.

### 9. Chapter 05's bisection method has six steps that must run in order. Which order matches?

- A. Reproduce offline → top-down walk → right run? → descend → bisect by attribute → failure signature.
- B. Right run? → failure signature → top-down walk → descend to the first suspect span → bisect by attribute → reproduce offline.
- C. Bisect by attribute → right run? → descend → reproduce offline → failure signature → top-down walk.
- D. Failure signature → reproduce offline → right run? → descend → top-down walk → bisect by attribute.

### 10. Chapter 05 names three anti-patterns that make the bisection workflow fail. Which of the following *is* one of them?

- A. Quoting attribute names and values verbatim from the viewer.
- B. Narrating the failure ("the model got confused") instead of quoting the span attribute that pins it.
- C. Running step 1 before step 6.
- D. Annotating the trace in the backend after you finish the walk.

### 11. Chapter 06 names three sampling positions that work together. Which description matches the chapter?

- A. Head sampling at the Collector; tail sampling at the SDK; signal-driven sampling at the vendor backend.
- B. Head sampling at the SDK (cheap, blind to the tail); tail sampling at the Collector (buffers all spans of a trace and keeps errors, slow traces, canary arms); signal-driven sampling that promotes traces based on an eval signal such as a judge score below threshold or a guardrail trip.
- C. There is only one position — the backend — because the other two are not supported by OpenTelemetry.
- D. Head and tail sampling are equivalent; pick either, not both.

### 12. Chapter 06 states an ordering rule for the Collector processor pipeline when both PII scrubbing and tail sampling are active. Which is it, and why?

- A. `tail_sampling → attributes → batch` — tail-sample first so you only scrub what you keep.
- B. `attributes → tail_sampling → batch` — scrub before tail-sample so raw PII never enters the tail-sampling buffer.
- C. The order does not matter; both processors operate on copies.
- D. `batch → attributes → tail_sampling` — batching first improves throughput and the other two run in parallel.

---

## Answer key

1. **D.** Chapter 01 lists the five debugger questions as: exact prompt / input; what the model returned in full; which retrieved chunks / tools / sub-agents were called in what order; cost and latency; user / session / feature-flag context for *this* run. "Which feature flag flipped most recently across the fleet" is a cross-trace fleet query, not a per-trace debugger question.
2. **A.** Chapter 01's "how LLM traces differ from a normal microservice trace" section leads with "the payload is the point" — a REST trace usually records that a call happened, not what it said, whereas an LLM trace needs the actual prompt and completion text. B, C, and D contradict chapter 01 (propagation is required, cost is first-class on LLM spans, TTFT/TPOT are distinct from wall-clock on streaming).
3. **B.** Chapter 02's practical guidance is "pick one convention as **primary per span** and stay consistent"; the dual-convention mirror pattern in the Example section shows annotating the span with both names explicitly when both are needed, with the primary documented in the schema. Mixing half and half on one span (C) is explicitly called out as making viewer rendering unreliable.
4. **B.** Chapter 02's "Content — attributes vs events" box: OTel GenAI historically used span events (`gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.system.message`, `gen_ai.choice`) for message content; OpenInference uses indexed attributes (`llm.input_messages.<i>.message.content`, etc.) on the span itself.
5. **B.** Chapter 03 names the two transports as SDK-direct and OTLP (over gRPC or HTTP), with the OTLP + OpenTelemetry Collector path being the portable escape hatch — "instrument once against OTel; the Collector runs a pipeline of receivers → processors → exporters; and you can add or remove backends by changing config, not code."
6. **A.** Chapter 03's decision guide lists Phoenix (self-host) and Langfuse (self-host) as the two purpose-built LLM-app options when on-prem / self-hosted is mandatory, with a Collector piping to Jaeger or Grafana Tempo as the vendor-neutral fallback.
7. **C.** Chapter 04's "User-id-or-hash" section: HMAC-SHA256 of the raw user id, keyed with a per-tenant salt held in a KMS; pseudonymous under GDPR Art. 4(5), so personal-data obligations still apply. The chapter explicitly warns against unsalted SHA256 (look-up-table attack) and against plaintext.
8. **B.** Chapter 04's content-granularity rule: "Full-fidelity content lives on the LLM span it belongs to. The CHAIN / AGENT parent gets a *redacted repro summary* on `output.value* — not the raw model text." The same rule applies to tool arguments (on the TOOL span) and retrieved documents (on the RETRIEVER span).
9. **B.** Chapter 05's "The bisection method" states six steps in order: confirm you are looking at the right run → read input and final output, name the failure signature → top-down walk over child span kinds → descend to the first suspect span → bisect by root attribute (`experiment.arm`, `service.version`, `gen_ai.response.model`) → reproduce the failure offline as a fixture.
10. **B.** Chapter 05's "Guardrails on the workflow itself" names three anti-patterns: narrating the failure instead of quoting the span attribute, fixing at the wrong scope, and skipping the reproducer step. "The model got confused" is explicitly the contrast given for the narration anti-pattern.
11. **B.** Chapter 06 names head sampling at the SDK (`ParentBased`, `TraceIdRatioBased`, `AlwaysOn`, `AlwaysOff` — cheap, no buffering, blind to the tail), tail sampling at the OpenTelemetry Collector (`tail_sampling` processor in `opentelemetry-collector-contrib` — buffers all spans and can keep errors, high-cost, canary, slow tails), and signal-driven sampling on an eval signal computed from the trace (judge score below threshold, guardrail trip).
12. **B.** Chapter 06's "A Collector recipe" section states the ordering rule explicitly: `attributes → tail_sampling → batch`. The reason is that the tail-sampling processor buffers raw traces in memory for the decision window; if a redaction bug ships payload downstream, the tail buffer is one of the places you should not have looked — "scrub before tail-sample" is the rule.
