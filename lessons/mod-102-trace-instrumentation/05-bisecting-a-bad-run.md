# Bisecting a Bad Run: The Trace-Viewer Workflow for Nested Agents

## Motivation

Every trace-instrumentation project pays for itself on one specific day: the day someone reports a bad output and hands you a trace id. What you do next is what tells whether the schema (chapter 04), the conventions (chapter 02), and the backend (chapter 03) actually pull their weight — or whether you fall back to re-running the request by hand in staging, guessing which prompt was in flight.

The workflow this chapter teaches is a **bisection**: start at the root of the trace, narrow to the smallest span that could have caused the failure, quote the attribute that pins it, and hand off a reproducer to CI (mod-106) and to the human-review queue (mod-109). It works because the schema encodes the causal graph. Done well, root-cause on an LLM-app bug moves from "an afternoon" to "twenty minutes."

## Core concepts

### The bisection method

Six steps, always in this order. Skipping any of them is where mistakes come from.

1. **Confirm you are looking at the right run.** Read the root-span attributes: `app.surface`, `session.id`, `user.id` (hashed), `service.version`, `experiment.arm`. One sentence: "This is the support_agent surface on 2026.07.08-a7f2, arm reply-model-v3, for session sess_9e4b… . The complaint matches this run." If any of these disagree with the bug report, stop — you have the wrong trace.

2. **Read the input and the final output.** Copy the root `input.value` and the root `output.value` into your report. Name the failure signature precisely: *wrong content*, *wrong tool call*, *wrong refusal*, *over-refusal*, *loop that never terminated*, *silent error*. That signature is what you bisect against — do not narrate.

3. **Walk the tree top-down before touching any span.** Read the sequence of child span kinds under the root. For the support agent: AGENT → LLM.plan → RETRIEVER → TOOL → LLM.reply → GUARDRAIL. Anomalies you can spot in this pass alone:
   - **Missing kind.** No RETRIEVER child when the surface always retrieves → retrieval was skipped.
   - **Extra kind.** Two TOOL children when policy allows one → an unexpected extra tool call.
   - **Repeated kind.** LLM.plan × 5 → planning loop.
   - **Wrong order.** GUARDRAIL before LLM.reply on a surface where guardrail is post-generation → mis-wired guardrail.
   - **Truncated tree.** Root ended early, no children of a kind you expect → early error.

4. **Descend to the first suspect span.** For each candidate: compare its `input.value` to its parent's context, and its `output.value` to what the next sibling (or child) saw as input. A trace obeys causality: the LLM.plan output feeds LLM.reply's input; if the two disagree, the runtime shim did something to the message in between. That is a real bug.

5. **Bisect by attribute.** Some failures are population-level, not span-level. Use the root attributes: filter across recent traces by `experiment.arm`, `service.version`, `gen_ai.response.model`. If the failure only occurs on `reply-model-v3` or only on `service.version=2026.07.08-a7f2`, that is your attribution.

6. **Reproduce the failure offline.** Serialise the smallest chain that reproduces: the raw `input.value`, the exact prompt template (from `llm.prompt_template.template` or its hash + the template store), the decoding parameters (`llm.invocation_parameters`), and the retrieved documents you observed. Re-run locally. If the bug reproduces, you have a fixture for mod-106; if it does not, the failure was environmental (data drift, upstream API, race) — a different investigation.

### Common failure signatures and where they land

| Signature | Where the tell is | What to check on the span |
|---|---|---|
| Wrong tool arguments | LLM.plan output | `llm.output_messages.<i>.message.content` and `llm.tool_calls.<i>.tool_call.function.arguments` — parse against `tool.parameters` schema |
| Retrieval missed the answer | RETRIEVER span | `retrieval.documents.<i>.document.score` and `.id` — is the ground-truth id present? |
| Ungrounded assertion | LLM.reply output vs RETRIEVER | Substring match of the asserted claim against `retrieval.documents.*.document.content` |
| Loop that never terminated | AGENT / CHAIN | Child span count vs a step-cap; look for a token cap that fired late (`gen_ai.response.finish_reasons=["length"]`) |
| Over-refusal | GUARDRAIL span | `app.guardrail_verdict=block` on benign input; check `guardrail.rule_hit` if you emit it |
| Silent tool failure | TOOL span | `status=Error`, `output.value` contains an error payload the parent did not surface |
| Model version drift | Root or LLM span | `gen_ai.response.model` != `gen_ai.request.model` — provider served a different pin |
| Wrong prompt in flight | LLM span | `llm.prompt_template.template` hash != what your repo's HEAD would produce |

Each row is falsifiable on a trace alone. That is the value of chapter 04's schema; it is why "prompt content on the LLM span it belongs to" is a rule, not a preference.

### Sub-traces and span links

Agents that spawn sub-agents produce **sub-traces** — either because the sub-agent runs in a different service (crossing a network boundary), or because you deliberately span-link across a queue or a scheduled retry.

Two mechanisms make sub-traces first-class:

- **W3C Trace Context propagation.** When the coordinator agent calls a sub-agent HTTP endpoint, pass `traceparent` and `tracestate` in the headers. The sub-agent's runtime picks them up, so its root span is a child of the coordinator's span even though they run in different services. This is the default for any OpenTelemetry-instrumented HTTP client / server.
- **Span links.** When causality is fan-out / join (a coordinator kicks off three async sub-agents, waits for one to return, and cancels the others), the "returned" sub-trace's root span carries an OTel **link** back to the coordinator span. The trace viewer follows the link even though the causal edge is not parent-child.

Both are ordinary OpenTelemetry features (see <https://opentelemetry.io/docs/specs/otel/trace/api/#link>). Use them so that the top-down walk from step 3 above does not stop at a service boundary.

### Annotate and hand off

A bad run that stops at the on-call's terminal is a bad run that repeats. Every trace-viewer in chapter 03 lets you attach an annotation, a label, or a score to a trace or a span (consult the vendor docs for the exact call). Use it:

- **Annotation** — one paragraph in English: which span held the root cause, what the failure signature was, what the reproducer is.
- **Label / tag** — machine-readable tag (`bug/tool-argument-format`, `bug/ungrounded-reply`, `regression/reply-model-v3`) — mod-109 routes on these.
- **Score** — a numeric SLI score attached to the span (`app.judge_score = 1` on a 5-point rubric) so the mod-107 online-eval loop counts this trace against the SLO.

Then **route** the trace: annotated → mod-109 human-review queue (label `needs_triage`), and reproducer JSON → mod-106 regression suite as a new fixture with a link back to the trace id.

### Guardrails on the workflow itself

Three anti-patterns to name and avoid:

- **Narrating the failure instead of quoting the span.** "The model got confused" is not a diagnosis. `llm.tool_calls.0.tool_call.function.arguments = "{\"order_id\": null}"` is.
- **Fixing at the wrong scope.** A bug that reproduces on a retriever unit test does not need an end-to-end E2E fix; propose the smallest test that would catch it.
- **Skipping the reproducer step.** A trace-based diagnosis without an offline reproducer is a guess. If the fixture does not reproduce, the trace did not tell you the whole story — go back and look for what the schema is missing (chapter 04 is a living document).

## Example: bisecting "Ticket opened on the wrong order"

The bug report:

> User `u_a41c…` complained that they asked about order **12345** but the agent opened a ticket referencing order **12354**. Trace id `9c8f42e0b5a7…`.

**Step 1 — right run?** Root span attributes:

```
app.surface       = "support_agent"
service.version   = "2026.07.08-a7f2"
experiment.arm    = "reply-model-v3"
user.id           = "u_a41c…"       ✔ matches
session.id        = "sess_2e4a…"
input.value       = "Where is my order 12345?"    ✔ matches complaint
output.value      = "Ticket #1263 opened for order 12354."   ✔ matches failure
```

Right run.

**Step 2 — failure signature.** Wrong tool argument: `order_id=12354` was passed to `open_ticket` when the user asked about `12345`. Not a refusal, not a retrieval miss, not an ungrounded reply. Digit transposition.

**Step 3 — top-down walk.** Child sequence: `llm.plan → retriever.docs.search → tool.open_ticket → llm.reply → guardrail.output_policy`. All expected kinds present, no repeats. Nothing structural to report.

**Step 4 — descend to the first suspect span.** The order id first appears in `llm.plan.output_messages` — that is where the LLM emitted the tool call payload. Read:

```
llm.plan.llm.output_messages.0.message.content = ""     (tool-call turn — no assistant text)
llm.plan.llm.tool_calls.0.tool_call.function.name       = "open_ticket"
llm.plan.llm.tool_calls.0.tool_call.function.arguments  = "{\"order_id\": \"12354\", \"reason\": \"no_shipping_update\"}"
```

The `12354` was in the model's output. Not the tool. Compare to `llm.plan.input_messages`:

```
llm.plan.llm.input_messages.1.message.role/content = "user" / "Where is my order 12345?"
```

The user said 12345. The plan LLM emitted 12354. This is a model-side digit transposition on the plan call — a class of failure common on numeric ids in prompts. Not a bug in your retrieval, not a bug in the tool.

Cross-check against the RETRIEVER span:

```
retriever.docs.search.input.value = "order 12345 status"      ← from the plan LLM's tool-call payload
```

Wait — the retriever query has 12345, the tool call has 12354. So the plan LLM emitted *two* outputs (retriever query and tool-call arguments) and got the digit wrong only in the tool-call arguments. That refines the root cause: the plan LLM produced inconsistent numeric ids across its outputs.

**Step 5 — bisect by attribute.** Filter last 200 traces on `experiment.arm = "reply-model-v3"` for numeric-id transpositions on `tool.open_ticket`. If the class of failure is arm-specific — the previous arm didn't do this — you have an attribution to the arm. (This step is the setup for a mod-107 online-eval scorer.)

**Step 6 — reproduce offline.** Save a fixture:

```jsonc
{
  "surface": "support_agent",
  "input": "Where is my order 12345?",
  "prompt_template_hash": "sha256:0f7e…",
  "invocation_parameters": {"temperature": 0.2, "max_tokens": 512},
  "expected_tool_call": {"name": "open_ticket", "arguments": {"order_id": "12345", "reason": "no_shipping_update"}}
}
```

Re-run against the same model version pin. If reproducing, add this fixture to the mod-106 regression suite under `fixtures/numeric_id_consistency/`. Propose the smallest scoped test: **plan-only test** — no retriever, no tool, no reply — asserting that the emitted tool-call `order_id` equals the user-provided id. A reply-level end-to-end test is unnecessary at this scope.

**Annotation and hand-off.** Attach an annotation to the trace on the backend: "Root cause: plan LLM digit transposition on tool-call arguments; retriever query was correct." Tag: `bug/tool-argument-format`. Route to mod-109 human review with `needs_triage` label to confirm the class, and log a `mod-107` follow-up: propose an online scorer that checks numeric id consistency across span outputs on every sampled trace.

## Summary

- Bisection is six steps in order: right run → failure signature → top-down walk → descend to suspect span → filter by root attributes → reproduce offline.
- Each step quotes an **attribute name and value** from the schema (chapter 04), not narrative about the model.
- Common failure signatures have canonical resting places in the tree — the schema places them there deliberately.
- Cross-service and fan-out cases are supported by W3C Trace Context propagation and OTel **span links**; sub-traces should stay reachable from the coordinator trace.
- Every diagnosed run produces three artefacts: a trace annotation on the backend, a reproducer fixture for mod-106, and a route into mod-109's review queue.
- Never fix at a larger scope than the smallest test that would catch the failure.

Chapter 06 addresses the sampling and PII policy that decides which traces you keep for exactly this workflow.
