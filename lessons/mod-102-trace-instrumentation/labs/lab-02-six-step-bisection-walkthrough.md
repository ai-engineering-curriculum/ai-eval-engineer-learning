# lab-02: Walk the Six-Step Bisection Against a Canned Bad Run

**Estimated duration:** 45–60 minutes
**Pairs with:** chapter 05 of this module, and chapters 01 / 04 for the schema shape the walk quotes against.

## Objective

Chapter 05 teaches the six-step bisection for root-causing a bad LLM-app run from the trace alone. The exercise for the chapter (exercise-04) asks you to seed and bisect three failures on *your own* instrumented agent — that is the full workout. This lab is the single-rep warm-up: one pre-canned bad-run trace, a small Python helper that extracts attributes without needing a vendor viewer, and a bisection report you write against chapter 05's exact six-step structure.

At the end you will have:

- A six-step bisection report that pins the root cause to a named attribute value.
- A reproducer fixture in the shape chapter 05 shows and `mod-106` will consume.
- A clear sense of which schema attribute you would add if the walk had stopped short — the same reflection `missed-signal.md` in exercise-04 asks for.

This lab does *not* duplicate exercise-04: the exercise seeds failures on your own agent across three distinct signatures and routes each hand-off into `mod-109` tickets and backend annotations. The lab hands you one canned trace and asks you to execute the method mechanically against it.

## Prerequisites

- Chapter 05 read front to back. Chapters 01 and 04 recommended because the walk quotes attribute names chapter 04 defines.
- Python 3.10+ (`python3 --version`).
- Lab-01 is not required but recommended — the trace shape in this lab mirrors the one lab-01's `support_agent.py` emits.
- A working directory you can write files into.

## Layout you will build

```
mod-102-lab-02/
  trace_bad.json
  bisect.py
  bisection-report.md
  fixtures/
    run-1.json
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-102-lab-02/fixtures && cd mod-102-lab-02
```

### Step 2 — Save the canned bad-run trace (`trace_bad.json`)

The bug is the one chapter 05 walks in its worked example: user asked about order **12345**, but the opened ticket references order **12354** — a digit transposition in the plan LLM's tool-call arguments. The retrieval query still has `12345`, which is how you pin the root cause to the plan span's tool-call output specifically.

Create `trace_bad.json` (this is the whole trace as a single JSON document — a list of span objects):

```json
[
  {
    "name": "agent.support_agent",
    "span_id": "01",
    "parent_span_id": null,
    "attributes": {
      "openinference.span.kind": "AGENT",
      "app.surface": "support_agent",
      "app.step": "agent.support_agent",
      "session.id": "sess_2e4a",
      "user.id": "u_a41c7b2c0f3e8d91",
      "service.name": "support_agent",
      "service.version": "2026.07.08-a7f2",
      "schema.version": "1.2.0",
      "experiment.arm": "reply-model-v3",
      "input.value": "Where is my order 12345?",
      "output.value": "Ticket #1263 opened for order 12354.",
      "app.cost_usd": 0.0021,
      "app.ttft_ms": 512
    }
  },
  {
    "name": "llm.plan",
    "span_id": "02",
    "parent_span_id": "01",
    "attributes": {
      "openinference.span.kind": "LLM",
      "app.step": "plan",
      "llm.provider": "openai",
      "llm.model_name": "gpt-4o-2024-08-06",
      "llm.input_messages.0.message.role": "system",
      "llm.input_messages.0.message.content": "You are a support planner.",
      "llm.input_messages.1.message.role": "user",
      "llm.input_messages.1.message.content": "Where is my order 12345?",
      "llm.output_messages.0.message.role": "assistant",
      "llm.output_messages.0.message.content": "",
      "llm.tool_calls.0.tool_call.function.name": "open_ticket",
      "llm.tool_calls.0.tool_call.function.arguments": "{\"order_id\": \"12354\", \"reason\": \"no_shipping_update\"}",
      "llm.token_count.prompt": 842,
      "llm.token_count.completion": 34,
      "llm.token_count.total": 876,
      "gen_ai.response.model": "gpt-4o-2024-08-06",
      "gen_ai.response.finish_reasons": ["tool_calls"],
      "app.cost_usd": 0.0009
    }
  },
  {
    "name": "retriever.docs.search",
    "span_id": "03",
    "parent_span_id": "01",
    "attributes": {
      "openinference.span.kind": "RETRIEVER",
      "app.step": "docs.retrieve",
      "app.retriever_name": "hybrid_bm25_dense",
      "app.top_k": 3,
      "input.value": "order 12345 status",
      "retrieval.documents.0.document.id": "doc-142",
      "retrieval.documents.0.document.score": 0.81,
      "retrieval.documents.0.document.content": "Order 12345 is pending carrier handoff.",
      "retrieval.documents.1.document.id": "doc-91",
      "retrieval.documents.1.document.score": 0.78,
      "retrieval.documents.1.document.content": "Orders without shipping update after 48h may be escalated."
    }
  },
  {
    "name": "tool.open_ticket",
    "span_id": "04",
    "parent_span_id": "01",
    "attributes": {
      "openinference.span.kind": "TOOL",
      "app.step": "open_ticket",
      "tool.name": "open_ticket",
      "tool.description": "Open a support ticket in Zendesk",
      "input.value": "{\"order_id\": \"12354\", \"reason\": \"no_shipping_update\"}",
      "output.value": "{\"ticket_id\": 1263, \"status\": \"open\"}",
      "status": "OK"
    }
  },
  {
    "name": "llm.reply",
    "span_id": "05",
    "parent_span_id": "01",
    "attributes": {
      "openinference.span.kind": "LLM",
      "app.step": "reply",
      "llm.provider": "openai",
      "llm.model_name": "gpt-4o-2024-08-06",
      "llm.output_messages.0.message.role": "assistant",
      "llm.output_messages.0.message.content": "Ticket #1263 opened for order 12354.",
      "llm.token_count.prompt": 1128,
      "llm.token_count.completion": 83,
      "llm.token_count.total": 1211,
      "gen_ai.response.model": "gpt-4o-2024-08-06",
      "gen_ai.response.finish_reasons": ["stop"],
      "app.cost_usd": 0.0012
    }
  },
  {
    "name": "guardrail.output_policy",
    "span_id": "06",
    "parent_span_id": "01",
    "attributes": {
      "openinference.span.kind": "GUARDRAIL",
      "app.step": "guardrail.output",
      "app.guardrail_id": "policy_v14",
      "app.guardrail_version": "14.2",
      "input.value": "Ticket #1263 opened for order 12354.",
      "output.value": "Ticket #1263 opened for order 12354.",
      "app.guardrail_verdict": "allow"
    }
  }
]
```

This file is the whole trace; in lab-01 you saw the same shape emitted by the SDK as six JSON-per-line objects. The flat-list shape here lets you load it with one `json.load`.

The bug report that comes with it:

> User `u_a41c…` complained that they asked about order **12345** but the agent opened a ticket referencing order **12354**. Trace id above. Session `sess_2e4a`.

### Step 3 — Write the bisection helper (`bisect.py`)

The helper does not *do* the bisection — it just answers the mechanical questions step 1 and step 3 of chapter 05 want you to answer first. You still write the report yourself.

```python
# bisect.py
"""Chapter 05 bisection helper.

Prints the AGENT root attributes (step 1), the ordered sequence of
child-span kinds and app.steps (step 3), and a couple of smoke checks
that catch the most common schema gaps — enough to steer a human
through the six-step walk without opening a vendor viewer."""

import argparse
import json
import sys
from pathlib import Path


def _attrs(span: dict) -> dict:
    return span.get("attributes") or {}


def _root(spans: list[dict]) -> dict:
    for s in spans:
        if _attrs(s).get("openinference.span.kind") == "AGENT":
            return s
    raise SystemExit("no AGENT root span in file")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("path", help="Path to a trace JSON file")
    args = parser.parse_args()
    spans = json.loads(Path(args.path).read_text())
    root = _root(spans)
    children = [s for s in spans if s.get("parent_span_id") == root["span_id"]]

    print("=== Step 1 — right run? (root attributes) ===")
    for k in (
        "app.surface",
        "session.id",
        "user.id",
        "service.version",
        "experiment.arm",
        "input.value",
        "output.value",
    ):
        print(f"  {k} = {_attrs(root).get(k)!r}")

    print()
    print("=== Step 3 — top-down walk (child span kinds, in order) ===")
    for i, s in enumerate(children):
        a = _attrs(s)
        print(
            f"  [{i}] name={s['name']!r} kind={a.get('openinference.span.kind')!r}"
            f" step={a.get('app.step')!r}"
        )

    # Smoke checks a human still has to interpret.
    print()
    print("=== Smoke checks ===")
    llm_plan = next(
        (s for s in children if _attrs(s).get("app.step") == "plan"), None
    )
    retriever = next(
        (s for s in children if _attrs(s).get("app.step") == "docs.retrieve"), None
    )
    tool = next(
        (s for s in children if _attrs(s).get("app.step") == "open_ticket"), None
    )
    if llm_plan:
        print(
            "  plan.tool_call.arguments =",
            _attrs(llm_plan).get("llm.tool_calls.0.tool_call.function.arguments"),
        )
    if retriever:
        print("  retriever.input.value   =", _attrs(retriever).get("input.value"))
    if tool:
        print("  tool.input.value        =", _attrs(tool).get("input.value"))
        print("  tool.output.value       =", _attrs(tool).get("output.value"))

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Run it:

```bash
python3 bisect.py trace_bad.json
```

**Expected output:** the root attributes block (step 1), the five child spans in order (step 3), and the three smoke-check lines. The two rows that matter for the specific bug:

```
  plan.tool_call.arguments = {"order_id": "12354", "reason": "no_shipping_update"}
  retriever.input.value   = order 12345 status
```

Two different numeric ids. That mismatch is the single most informative line the helper will print — chapter 05 §Example calls it out as the root-cause tell for this failure class.

### Step 4 — Execute the six steps and write `bisection-report.md`

You write the report from the data `bisect.py` just printed and the trace JSON. The structure is chapter 05's six headings verbatim. Target ~250 words total; each section quotes an attribute value, not narrative.

Create `bisection-report.md`:

```markdown
# Bisection report — run-1

Bug report: user asked about order 12345; agent opened ticket referencing order 12354.

## Step 1 — right run?

- `app.surface = "support_agent"` matches the surface in the bug report.
- `session.id = "sess_2e4a"` matches the complaint's session id.
- `user.id = "u_a41c7b2c0f3e8d91"` matches the hashed user id in the complaint.
- `input.value = "Where is my order 12345?"` matches the complaint's query.
- `output.value = "Ticket #1263 opened for order 12354."` matches the failure.

This is the right run.

## Step 2 — failure signature

**Wrong tool arguments.** The ticket was opened for order 12354 but the user asked about order 12345. Not a refusal, not a retrieval miss, not an ungrounded reply. Digit transposition of the numeric id somewhere in the chain.

## Step 3 — top-down walk

Child-kind sequence (from `bisect.py`):

1. LLM `plan`
2. RETRIEVER `docs.retrieve`
3. TOOL `open_ticket`
4. LLM `reply`
5. GUARDRAIL `guardrail.output`

All expected kinds present, no repeats, no reordering. Nothing structural to flag — the anomaly must be inside a span, not in the tree shape.

## Step 4 — descend to suspect span

The order id first appears in two outputs of the plan LLM, which is the earliest place the model could have introduced the bad digits:

- `llm.plan.llm.tool_calls.0.tool_call.function.arguments = "{\"order_id\": \"12354\", ...}"` — **wrong** (12354).
- `retriever.docs.search.input.value = "order 12345 status"` — **right** (12345).

The plan LLM emitted *both* outputs on the same call, and only the tool-call-arguments output had the transposition. That rules out the retriever and the tool as the source: the TOOL span received `12354` as `input.value` and faithfully opened a ticket for it; the RETRIEVER received `12345` and ran the right query.

**Smallest suspect span:** `llm.plan` — specifically the `llm.tool_calls.0.tool_call.function.arguments` attribute.

## Step 5 — bisect by attribute

Hypothesis: the failure class is `experiment.arm = "reply-model-v3"`-specific (chapter 05's step-5 pattern on this exact trace). On a real deployment I would filter the last 200 traces by that attribute and count numeric-id transpositions on `tool.open_ticket.input.value` vs the user's `input.value`. The lab does not have 200 traces, so this step records the filter I would run and the comparison trace I would seek:

- Filter: `experiment.arm == "reply-model-v3"` AND `tool.name == "open_ticket"` over the last 24h.
- Compare against the same filter for the previous arm (`reply-model-v2`).
- If the transposition class is only on the current arm, that is the attribution.

## Step 6 — reproduce offline

Saved as `fixtures/run-1.json` (see next step). Intent: re-run the plan LLM against the same model pin, the same prompt template hash, the same decoding parameters, and assert `tool_call.function.arguments.order_id == "12345"`. Reply, retriever, and tool are out of scope — the smallest reproducer for a plan-LLM bug is a plan-only test (chapter 05 §Example, scope guardrail).
```

### Step 5 — Save the reproducer fixture (`fixtures/run-1.json`)

Chapter 05 §Step 6 shows the fixture shape. The fields are small on purpose — this is what `mod-106`'s regression suite will consume later.

Create `fixtures/run-1.json`:

```json
{
  "surface": "support_agent",
  "trace_id": "9c8f42e0b5a7",
  "input": "Where is my order 12345?",
  "llm_model": "gpt-4o-2024-08-06",
  "prompt_template_hash": "sha256:0f7e-stub",
  "invocation_parameters": {"temperature": 0.2, "max_tokens": 512},
  "expected_tool_call": {
    "name": "open_ticket",
    "arguments": {"order_id": "12345", "reason": "no_shipping_update"}
  },
  "failure_signature": "wrong_tool_arguments",
  "reproducer_scope": "plan_only",
  "notes": "Plan-LLM digit transposition on tool-call arguments; retriever query was correct."
}
```

### Step 6 — Sanity-check the report against chapter 05

Re-read chapter 05 §Guardrails on the workflow itself. For each anti-pattern, confirm your report does not commit it.

- **Narrating the failure** — step 4 quotes `llm.tool_calls.0.tool_call.function.arguments` verbatim, not "the model got confused." ✓
- **Fixing at the wrong scope** — `reproducer_scope = "plan_only"` names the smallest test that would catch the failure. ✓
- **Skipping the reproducer step** — `fixtures/run-1.json` exists on disk. ✓

If any of the three are not true for your report, rewrite that section before continuing.

### Step 7 — Name one missing attribute you wished were on the trace

Chapter 05 and exercise-04 both call out that every bisection usually exposes one attribute the schema does not yet carry. For *this* trace, a worked example: there is no attribute recording *which prompt version* the plan LLM ran under. `llm.prompt_template.template` is absent on `llm.plan`, so the bisection cannot filter "this failure class only shows up when `prompt_template.hash == sha256:0f7e...`". That is the schema patch you would file.

Append one paragraph to `bisection-report.md`:

```markdown
## Missed signal

The plan span does not carry `llm.prompt_template.template` or
`app.prompt_template.hash`. Without either, step 5 cannot bisect by
prompt version — if this failure were prompt-version-specific, a
reviewer could not prove it from the trace. Schema patch: add
`llm.prompt_template.template` (or a stable hash) as a MUST on the LLM
kind. Matches chapter 04 §Per-kind minimum attribute sets.
```

If you can see a different missing attribute, use that one — one named gap is enough.

### Step 8 — Checkpoint

You are done with this lab when every statement below is true.

- [ ] `python3 bisect.py trace_bad.json` prints the root attributes, the five child spans in order, and the three smoke-check lines.
- [ ] `bisection-report.md` contains six sections named exactly after chapter 05's six steps, in order.
- [ ] Every attribute reference in the report is quoted *verbatim* — attribute name and value — and no section narrates without quoting.
- [ ] `fixtures/run-1.json` exists and contains at least `surface`, `input`, `llm_model`, `invocation_parameters`, and `expected_tool_call`.
- [ ] The report has a `reproducer_scope` statement that names the smallest test (plan-only, not end-to-end).
- [ ] You have named at least one missed-signal attribute in the final section.

## Where this hands off

- The bisection workflow is now muscle-memory; exercise-04 asks you to run it three more times against seeded failures on your own agent and route the outputs into `mod-109` tickets + backend annotations.
- `fixtures/run-1.json` is the fixture shape `mod-106` will consume into its regression suite. When `mod-106` gates the CI merge, the shape of the fixture is what the gate reads.
- `bisect.py` is a deliberately tiny stand-in for the vendor viewer. In production your on-call opens Phoenix / Langfuse / Weave / Braintrust / LangSmith and executes the same six steps clicking through the UI — chapter 03 exists so that experience is cheap.
- The missed-signal paragraph is the input `mod-110`'s eval-data platform will use to bump `schema.version` additively; chapter 04 §Versioning the schema is the rule for how to do that bump.
