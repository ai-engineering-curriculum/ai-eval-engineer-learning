# lab-01: Stand Up the Instrumented Reference App

**Estimated duration:** 60–90 minutes
**Pairs with:** chapters 01, 02, and 04 of this module.

## Objective

Build the minimum instrumented reference app the rest of this track consumes: a tiny support-agent surface with a plan LLM, a retriever, one tool, a reply LLM, and an output guardrail, emitting **OpenInference** span kinds and attributes on every step. The exporter is the OpenTelemetry `ConsoleSpanExporter`, which prints the whole trace tree to stdout as JSON — so this lab runs on a laptop with no vendor backend, no Docker, and no API keys.

At the end you can hand the printed trace to chapter 01's five debugger questions and answer every one of them without opening a viewer.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-01 writes the same app *twice* (OTel GenAI vs OpenInference) against a real model API and compares the shapes in a viewer.
- Exercise-02 fans out via a Collector to two real backends and runs a scorecarded drill.
- Exercise-03 writes the schema document for a surface you own.
- Exercise-04 bisects three seeded failures against an instrumented service.
- Exercise-05 writes the sampling + PII policy and runs a Collector drill.

This lab gives you a working toy app whose trace shape matches chapter 04's worked example — the artefact labs-02 and 03 extend, and the artefact `mod-106` CI gates will later enforce the schema MUSTs on.

## Prerequisites

- Chapters 01 (vocabulary and five debugger questions), 02 (OpenInference attributes), and 04 (per-kind MUSTs and nine root attributes) read.
- Python 3.10+ on `$PATH` (`python3 --version`).
- A working directory you can write files into. No internet, no model API keys, no Docker — the app stubs the model, retriever, tool, and guardrail so you focus on the instrumentation shape.

## Layout you will build

```
mod-102-lab-01/
  requirements.txt
  support_agent.py
  trace_check.py
  trace.jsonl           # written by the first run
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-102-lab-01 && cd mod-102-lab-01
```

### Step 2 — Install the OpenTelemetry Python SDK

The app only needs the API and SDK; the ConsoleSpanExporter ships with the SDK.

Create `requirements.txt`:

```
opentelemetry-api>=1.25.0
opentelemetry-sdk>=1.25.0
```

Install:

```bash
pip install -r requirements.txt
```

**Expected:** two packages install cleanly. You do not need `opentelemetry-exporter-otlp` for this lab; labs 02 and 03 pick that up.

### Step 3 — Write the stubbed surface and the instrumentation (`support_agent.py`)

The surface mirrors chapter 04's worked support-agent example:

```
agent.support_agent
├─ llm.plan             (LLM, model emits a tool call)
├─ retriever.docs.search (RETRIEVER, 3 ranked docs)
├─ tool.open_ticket     (TOOL, HTTP stub)
├─ llm.reply            (LLM, final assistant text)
└─ guardrail.output_policy (GUARDRAIL, verdict)
```

Every span carries the MUSTs from chapter 04's per-kind tables and the nine root attributes on the AGENT root. The model, retriever, tool, and guardrail are all deterministic stubs so the trace is stable from run to run.

Create `support_agent.py`:

```python
# support_agent.py
"""Stubbed support-agent surface instrumented with OpenInference attributes.

Chapter 04's worked example in code form. The model, retriever, tool, and
guardrail are all hand-rolled stubs — the point of this lab is the
instrumentation, not the LLM. ConsoleSpanExporter prints each span as
JSON so you can read the trace tree without a viewer."""

import hashlib
import hmac
import json
import sys
import time
from pathlib import Path

from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter,
)
from opentelemetry.trace import SpanKind, StatusCode, Status


SURFACE = "support_agent"
SERVICE_VERSION = "2026.11.25-lab01"
SCHEMA_VERSION = "1.0.0"
EXPERIMENT_ARM = "reply-model-v3"
MODEL_NAME = "stub-model-2026-11-25"
PROVIDER = "lab-stub"
TENANT_SALT = b"lab-salt-rotate-in-prod"  # never ship this value in prod.
TRACE_OUT = Path("trace.jsonl")


def _configure_tracer() -> None:
    resource = Resource.create(
        {
            "service.name": SURFACE,
            "service.version": SERVICE_VERSION,
        }
    )
    provider = TracerProvider(resource=resource)
    # Write each span as one JSON object to trace.jsonl so trace_check.py
    # can read it back. ConsoleSpanExporter prints to the file handle we
    # pass it, one JSON doc per span.
    out_handle = TRACE_OUT.open("w")
    # ConsoleSpanExporter pretty-prints by default (indent=4), which
    # breaks one-JSON-per-line. Override the formatter so trace_check.py
    # can read trace.jsonl line by line.
    provider.add_span_processor(
        BatchSpanProcessor(
            ConsoleSpanExporter(
                out=out_handle,
                formatter=lambda span: span.to_json(indent=None) + "\n",
            )
        )
    )
    trace.set_tracer_provider(provider)


def _user_hash(raw_user_id: str) -> str:
    mac = hmac.new(TENANT_SALT, raw_user_id.encode("utf-8"), hashlib.sha256)
    return "u_" + mac.hexdigest()[:16]


def _plan(user_msg: str) -> dict:
    # Stand-in for the plan LLM. Produces a tool-call payload.
    return {
        "assistant_content": "",
        "tool_call": {
            "name": "open_ticket",
            "arguments": {
                "order_id": "12345",
                "reason": "no_shipping_update",
            },
        },
        "finish_reason": "tool_calls",
        "input_tokens": 842,
        "output_tokens": 34,
    }


def _retrieve(query: str) -> list[dict]:
    # Stand-in for a vector + BM25 retriever. Three ranked docs.
    return [
        {
            "id": "doc-142",
            "score": 0.81,
            "content": "Order 12345 is pending carrier handoff.",
            "metadata": {"source": "kb/orders", "version": "2026-06-30"},
        },
        {
            "id": "doc-91",
            "score": 0.78,
            "content": "Orders without a shipping update after 48h may be escalated.",
            "metadata": {"source": "kb/policy", "version": "2026-05-12"},
        },
        {
            "id": "doc-208",
            "score": 0.55,
            "content": "Customer-facing FAQ on shipping windows.",
            "metadata": {"source": "kb/faq", "version": "2026-04-02"},
        },
    ]


def _tool_open_ticket(arguments: dict) -> dict:
    # Stand-in for the Zendesk HTTP tool.
    return {"ticket_id": 1247, "status": "open"}


def _reply(context_docs: list[dict], ticket: dict) -> dict:
    assistant = (
        "Ticket #1247 has been opened for your order 12345. "
        "You will get a shipping update within one business day."
    )
    return {
        "assistant_content": assistant,
        "finish_reason": "stop",
        "input_tokens": 1128,
        "output_tokens": 83,
    }


def _guardrail(reply_text: str) -> dict:
    # Stand-in for an output guardrail. Verdict is 'allow' for clean text.
    return {"verdict": "allow", "output": reply_text}


def _usd_cost(input_tokens: int, output_tokens: int) -> float:
    # Fictitious unit prices. Only the arithmetic matters.
    return round(input_tokens * 0.000001 + output_tokens * 0.000003, 6)


def run(raw_user_id: str, session_id: str, user_msg: str) -> str:
    tracer = trace.get_tracer("mod-102.lab-01.support_agent")

    with tracer.start_as_current_span(
        "agent.support_agent",
        kind=SpanKind.INTERNAL,
        attributes={
            "openinference.span.kind": "AGENT",
            "app.surface": SURFACE,
            "app.step": "agent.support_agent",
            "session.id": session_id,
            "user.id": _user_hash(raw_user_id),
            "service.name": SURFACE,
            "service.version": SERVICE_VERSION,
            "schema.version": SCHEMA_VERSION,
            "experiment.arm": EXPERIMENT_ARM,
            "input.value": user_msg,
            "input.mime_type": "text/plain",
        },
    ) as agent_span:
        start = time.monotonic()

        # --- plan LLM ----------------------------------------------------
        with tracer.start_as_current_span(
            "llm.plan",
            kind=SpanKind.CLIENT,
            attributes={
                "openinference.span.kind": "LLM",
                "app.step": "plan",
                "llm.provider": PROVIDER,
                "llm.model_name": MODEL_NAME,
                "llm.invocation_parameters": json.dumps(
                    {"temperature": 0.2, "max_tokens": 512}
                ),
                "llm.prompt_template.template": "You are a support planner...",
                "llm.input_messages.0.message.role": "system",
                "llm.input_messages.0.message.content": "You are a support planner.",
                "llm.input_messages.1.message.role": "user",
                "llm.input_messages.1.message.content": user_msg,
                # OTel GenAI mirror for interop.
                "gen_ai.system": PROVIDER,
                "gen_ai.request.model": MODEL_NAME,
            },
        ) as plan_span:
            plan = _plan(user_msg)
            plan_span.set_attributes(
                {
                    "llm.output_messages.0.message.role": "assistant",
                    "llm.output_messages.0.message.content": plan["assistant_content"],
                    "llm.tool_calls.0.tool_call.function.name": plan["tool_call"]["name"],
                    "llm.tool_calls.0.tool_call.function.arguments": json.dumps(
                        plan["tool_call"]["arguments"]
                    ),
                    "llm.token_count.prompt": plan["input_tokens"],
                    "llm.token_count.completion": plan["output_tokens"],
                    "llm.token_count.total": plan["input_tokens"] + plan["output_tokens"],
                    "gen_ai.response.model": MODEL_NAME,
                    "gen_ai.response.finish_reasons": [plan["finish_reason"]],
                    "gen_ai.usage.input_tokens": plan["input_tokens"],
                    "gen_ai.usage.output_tokens": plan["output_tokens"],
                    "app.cost_usd": _usd_cost(plan["input_tokens"], plan["output_tokens"]),
                }
            )

        # --- retriever ---------------------------------------------------
        retrieval_query = plan["tool_call"]["arguments"]["order_id"] + " status"
        with tracer.start_as_current_span(
            "retriever.docs.search",
            kind=SpanKind.CLIENT,
            attributes={
                "openinference.span.kind": "RETRIEVER",
                "app.step": "docs.retrieve",
                "app.retriever_name": "hybrid_bm25_dense",
                "app.top_k": 3,
                "input.value": retrieval_query,
            },
        ) as retriever_span:
            docs = _retrieve(retrieval_query)
            for i, d in enumerate(docs):
                retriever_span.set_attribute(
                    f"retrieval.documents.{i}.document.id", d["id"]
                )
                retriever_span.set_attribute(
                    f"retrieval.documents.{i}.document.score", d["score"]
                )
                retriever_span.set_attribute(
                    f"retrieval.documents.{i}.document.content", d["content"]
                )
                retriever_span.set_attribute(
                    f"retrieval.documents.{i}.document.metadata",
                    json.dumps(d["metadata"]),
                )

        # --- tool --------------------------------------------------------
        with tracer.start_as_current_span(
            "tool.open_ticket",
            kind=SpanKind.CLIENT,
            attributes={
                "openinference.span.kind": "TOOL",
                "app.step": "open_ticket",
                "tool.name": "open_ticket",
                "tool.description": "Open a support ticket in Zendesk",
                "input.value": json.dumps(plan["tool_call"]["arguments"]),
                "input.mime_type": "application/json",
            },
        ) as tool_span:
            tool_result = _tool_open_ticket(plan["tool_call"]["arguments"])
            tool_span.set_attribute("output.value", json.dumps(tool_result))
            tool_span.set_attribute("output.mime_type", "application/json")
            tool_span.set_status(Status(StatusCode.OK))

        # --- reply LLM ---------------------------------------------------
        with tracer.start_as_current_span(
            "llm.reply",
            kind=SpanKind.CLIENT,
            attributes={
                "openinference.span.kind": "LLM",
                "app.step": "reply",
                "llm.provider": PROVIDER,
                "llm.model_name": MODEL_NAME,
                "llm.invocation_parameters": json.dumps(
                    {"temperature": 0.2, "max_tokens": 512}
                ),
                "gen_ai.system": PROVIDER,
                "gen_ai.request.model": MODEL_NAME,
            },
        ) as reply_span:
            reply = _reply(docs, tool_result)
            reply_span.set_attributes(
                {
                    "llm.output_messages.0.message.role": "assistant",
                    "llm.output_messages.0.message.content": reply["assistant_content"],
                    "llm.token_count.prompt": reply["input_tokens"],
                    "llm.token_count.completion": reply["output_tokens"],
                    "llm.token_count.total": reply["input_tokens"] + reply["output_tokens"],
                    "gen_ai.response.model": MODEL_NAME,
                    "gen_ai.response.finish_reasons": [reply["finish_reason"]],
                    "gen_ai.usage.input_tokens": reply["input_tokens"],
                    "gen_ai.usage.output_tokens": reply["output_tokens"],
                    "app.cost_usd": _usd_cost(reply["input_tokens"], reply["output_tokens"]),
                }
            )

        # --- guardrail ---------------------------------------------------
        with tracer.start_as_current_span(
            "guardrail.output_policy",
            kind=SpanKind.INTERNAL,
            attributes={
                "openinference.span.kind": "GUARDRAIL",
                "app.step": "guardrail.output",
                "app.guardrail_id": "policy_v14",
                "app.guardrail_version": "14.2",
                "input.value": reply["assistant_content"],
            },
        ) as guard_span:
            verdict = _guardrail(reply["assistant_content"])
            guard_span.set_attribute("output.value", verdict["output"])
            guard_span.set_attribute("app.guardrail_verdict", verdict["verdict"])

        # --- root roll-ups ----------------------------------------------
        total_cost = _usd_cost(
            plan["input_tokens"] + reply["input_tokens"],
            plan["output_tokens"] + reply["output_tokens"],
        )
        elapsed_ms = int((time.monotonic() - start) * 1000)
        agent_span.set_attributes(
            {
                "output.value": f"Reply drafted; ticket #{tool_result['ticket_id']} opened.",
                "output.mime_type": "text/plain",
                "app.cost_usd": total_cost,
                "app.ttft_ms": elapsed_ms,  # stub — no streaming in this lab.
            }
        )
        return verdict["output"]


def main() -> int:
    _configure_tracer()
    reply = run(
        raw_user_id="alice@example.com",
        session_id="sess_lab01",
        user_msg="Where is my order 12345?",
    )
    # Force the batch processor to flush before exit so trace.jsonl is
    # complete when the script returns.
    trace.get_tracer_provider().shutdown()
    print(reply, file=sys.stderr)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Run it:

```bash
python3 support_agent.py
```

**Expected output:** the assistant text (`"Ticket #1247 has been opened..."`) is printed to *stderr*; the trace tree is written to `trace.jsonl` (one JSON object per span, six objects total).

Sanity-check the file length:

```bash
wc -l trace.jsonl
```

**Expected:** exactly `6 trace.jsonl` — one line each for `agent.support_agent`, `llm.plan`, `retriever.docs.search`, `tool.open_ticket`, `llm.reply`, `guardrail.output_policy`.

### Step 4 — Write the trace checker (`trace_check.py`)

The checker answers chapter 01's five debugger questions by reading `trace.jsonl` back in. If any question cannot be answered, the schema is incomplete — the same CI-friendly shape `mod-106` will build on.

```python
# trace_check.py
"""Reads trace.jsonl written by support_agent.py and verifies the trace
answers chapter 01's five debugger questions. Prints a one-line verdict
per question and exits non-zero if any fails."""

import json
import sys
from pathlib import Path


def _spans() -> list[dict]:
    rows = []
    for line in Path("trace.jsonl").read_text().splitlines():
        if not line.strip():
            continue
        rows.append(json.loads(line))
    return rows


def _root(spans: list[dict]) -> dict:
    # The AGENT root has openinference.span.kind == "AGENT".
    for s in spans:
        attrs = s.get("attributes", {}) or {}
        if attrs.get("openinference.span.kind") == "AGENT":
            return s
    raise SystemExit("no AGENT span in trace.jsonl")


def _by_kind(spans: list[dict], kind: str) -> list[dict]:
    return [
        s for s in spans
        if (s.get("attributes") or {}).get("openinference.span.kind") == kind
    ]


def _check(name: str, ok: bool, detail: str) -> bool:
    status = "OK" if ok else "FAIL"
    print(f"[{status}] {name}: {detail}")
    return ok


def main() -> int:
    spans = _spans()
    root = _root(spans)
    root_attrs = root.get("attributes", {}) or {}

    llm_spans = _by_kind(spans, "LLM")
    retriever_spans = _by_kind(spans, "RETRIEVER")
    tool_spans = _by_kind(spans, "TOOL")

    plan_llm = next(
        (s for s in llm_spans if (s["attributes"] or {}).get("app.step") == "plan"),
        None,
    )
    reply_llm = next(
        (s for s in llm_spans if (s["attributes"] or {}).get("app.step") == "reply"),
        None,
    )

    ok = True
    # Q1 — exact prompt
    ok &= _check(
        "Q1 exact prompt on plan LLM",
        bool(plan_llm and (plan_llm["attributes"] or {}).get(
            "llm.input_messages.1.message.content"
        )),
        "llm.input_messages.1.message.content present on llm.plan",
    )
    # Q2 — exact response
    ok &= _check(
        "Q2 exact response on reply LLM",
        bool(reply_llm and (reply_llm["attributes"] or {}).get(
            "llm.output_messages.0.message.content"
        )),
        "llm.output_messages.0.message.content present on llm.reply",
    )
    # Q3 — order of children: expect plan, retriever, tool, reply, guardrail
    child_steps = [
        (s["attributes"] or {}).get("app.step")
        for s in sorted(spans, key=lambda s: s.get("start_time", "") or "")
        if (s["attributes"] or {}).get("openinference.span.kind") != "AGENT"
    ]
    expected_steps = [
        "plan",
        "docs.retrieve",
        "open_ticket",
        "reply",
        "guardrail.output",
    ]
    ok &= _check(
        "Q3 child span order",
        child_steps == expected_steps,
        f"observed {child_steps} expected {expected_steps}",
    )
    # Q4 — cost and tokens
    has_tokens = all(
        (s["attributes"] or {}).get("llm.token_count.total") for s in llm_spans
    )
    has_cost = bool(root_attrs.get("app.cost_usd"))
    ok &= _check(
        "Q4 cost and tokens",
        has_tokens and has_cost,
        f"tokens on both LLM spans={has_tokens}, app.cost_usd on root={has_cost}",
    )
    # Q5 — user / session / arm context
    needed = ["session.id", "user.id", "service.version", "experiment.arm"]
    missing = [k for k in needed if not root_attrs.get(k)]
    ok &= _check(
        "Q5 session / user / arm on root",
        not missing,
        f"missing={missing or 'none'}",
    )
    # Bonus — user.id is hashed, not raw.
    uid = str(root_attrs.get("user.id", ""))
    ok &= _check(
        "Bonus user.id hashed",
        uid.startswith("u_") and "@" not in uid,
        f"user.id={uid}",
    )
    # Bonus — retriever carries doc ids + content + score (chapter 04 MUSTs).
    retriever = retriever_spans[0] if retriever_spans else None
    r_attrs = retriever["attributes"] if retriever else {}
    has_retriever_musts = bool(
        r_attrs.get("retrieval.documents.0.document.id")
        and r_attrs.get("retrieval.documents.0.document.content")
        and r_attrs.get("retrieval.documents.0.document.score") is not None
    )
    ok &= _check(
        "Bonus RETRIEVER MUSTs",
        has_retriever_musts,
        "id + content + score on retrieval.documents.0",
    )
    # Bonus — tool carries name + input.value + output.value.
    tool = tool_spans[0] if tool_spans else None
    t_attrs = tool["attributes"] if tool else {}
    has_tool_musts = bool(
        t_attrs.get("tool.name")
        and t_attrs.get("input.value")
        and t_attrs.get("output.value")
    )
    ok &= _check(
        "Bonus TOOL MUSTs",
        has_tool_musts,
        "tool.name + input.value + output.value",
    )

    return 0 if ok else 1


if __name__ == "__main__":
    sys.exit(main())
```

Run it:

```bash
python3 trace_check.py
echo "exit=$?"
```

**Expected output:**

```
[OK] Q1 exact prompt on plan LLM: llm.input_messages.1.message.content present on llm.plan
[OK] Q2 exact response on reply LLM: llm.output_messages.0.message.content present on llm.reply
[OK] Q3 child span order: observed ['plan', 'docs.retrieve', 'open_ticket', 'reply', 'guardrail.output'] expected ['plan', 'docs.retrieve', 'open_ticket', 'reply', 'guardrail.output']
[OK] Q4 cost and tokens: tokens on both LLM spans=True, app.cost_usd on root=True
[OK] Q5 session / user / arm on root: missing=none
[OK] Bonus user.id hashed: user.id=u_...
[OK] Bonus RETRIEVER MUSTs: id + content + score on retrieval.documents.0
[OK] Bonus TOOL MUSTs: tool.name + input.value + output.value
exit=0
```

### Step 5 — Break the schema and watch a MUST fail

Chapter 04 says the per-kind MUST column is the one CI enforces. Simulate a regression that violates it.

Open `support_agent.py`. In the RETRIEVER block, delete this line:

```python
                retriever_span.set_attribute(
                    f"retrieval.documents.{i}.document.content", d["content"]
                )
```

Re-run both scripts:

```bash
python3 support_agent.py
python3 trace_check.py
echo "exit=$?"
```

**Expected:** the `Bonus RETRIEVER MUSTs` line flips to `FAIL` and the exit code is `1`. That is the shape of a `mod-106` preflight check: the schema MUSTs are a machine-readable contract, and a regression that drops a MUST fails the build for the right reason. Restore the line before moving on.

### Step 6 — Map each span to chapter 04's MUST / SHOULD columns

Open `trace.jsonl` (`cat trace.jsonl | python3 -m json.tool | less`, or similar) and point at each span. For every span, state which chapter-04 per-kind table applies and name at least one MUST you can see in the JSON. One suggested mapping you should be able to defend:

| Span name | Kind | MUST attribute you can point at in the JSON |
|---|---|---|
| `agent.support_agent` | AGENT (chapter 04 §AGENT table) | `app.surface`, `session.id`, `schema.version`, `input.value`, `output.value` |
| `llm.plan` | LLM (chapter 04 §LLM table) | `llm.model_name`, `llm.input_messages.*`, `llm.token_count.total`, `gen_ai.response.finish_reasons` |
| `retriever.docs.search` | RETRIEVER (chapter 04 §RETRIEVER table) | `retrieval.documents.<i>.document.{id,score,content}` |
| `tool.open_ticket` | TOOL (chapter 04 §TOOL table) | `tool.name`, `input.value`, `output.value`, `status` |
| `llm.reply` | LLM | same as `llm.plan` |
| `guardrail.output_policy` | GUARDRAIL (chapter 04 names `guardrail.verdict` / `app.guardrail_verdict` as MUST) | `app.guardrail_verdict` |

If any row in this table is not satisfied by your trace, that is the schema change you need before the lab is really done.

### Step 7 — Verify the dual-convention mirror on the LLM spans

Chapter 02 names the dual-convention mirror pattern: pick one convention as primary per span and, where interop matters, deliberately mirror the same value under the other name. On `llm.plan` and `llm.reply`, confirm the following pairs hold the same value:

- `llm.provider` ⇔ `gen_ai.system`
- `llm.model_name` ⇔ `gen_ai.response.model`
- `llm.token_count.prompt` ⇔ `gen_ai.usage.input_tokens`
- `llm.token_count.completion` ⇔ `gen_ai.usage.output_tokens`

A one-off grep is enough:

```bash
python3 - <<'PY'
import json
for line in open("trace.jsonl"):
    s = json.loads(line)
    a = s.get("attributes") or {}
    if a.get("openinference.span.kind") != "LLM":
        continue
    pairs = [
        ("llm.provider", "gen_ai.system"),
        ("llm.model_name", "gen_ai.response.model"),
        ("llm.token_count.prompt", "gen_ai.usage.input_tokens"),
        ("llm.token_count.completion", "gen_ai.usage.output_tokens"),
    ]
    print(s["name"], {p: (a.get(p[0]), a.get(p[1])) for p in pairs})
PY
```

**Expected:** each pair carries the same value on each LLM span. Chapter 02's rule — "both names hold the same value" — is visible on the printed line.

### Step 8 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `python3 support_agent.py` produces `trace.jsonl` with exactly six spans.
- [ ] `python3 trace_check.py` prints `OK` for Q1–Q5 and both bonus checks and exits `0`.
- [ ] Deleting the `retrieval.documents.<i>.document.content` line makes the Bonus RETRIEVER MUSTs check fail and the exit code `1`.
- [ ] For each span, you can point at the chapter-04 MUST attribute(s) by name in `trace.jsonl`.
- [ ] On every LLM span the four OpenInference / OTel GenAI mirror pairs carry the same value.
- [ ] `user.id` on the AGENT root starts with `u_` and does NOT contain the raw email — chapter 04's hashing rule holds.

## Where this hands off

- `support_agent.py` is the reference app lab-02 bisects a seeded failure against and lab-03 runs drill traffic through a Collector in front of.
- `trace_check.py` is the shape `mod-106` will generalise into a CI MUST-column enforcer against a sampled set of real traces.
- The `schema.version = 1.0.0` attribute on the root is what `mod-110`'s eval-data platform will key against when the schema evolves. Chapter 04 §Versioning the schema is the rule you follow when you first bump this number.
- The ConsoleSpanExporter is deliberately the only exporter. Exercise-02 and lab-03 swap it for an OTLP exporter against a real Collector; the instrumentation code does not change — that is the whole point of instrumenting against OTel once.
