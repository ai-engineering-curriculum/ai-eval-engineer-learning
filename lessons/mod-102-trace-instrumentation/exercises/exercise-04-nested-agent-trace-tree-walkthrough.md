# exercise-04: Nested-Agent Trace-Tree Walkthrough

**Estimated effort:** 3 hours

## Objective

Practise the six-step bisection from chapter 05 against three real bad runs on a real nested-agent trace. Produce a **bisection report** per run that quotes attribute values from the schema — no narrative — and route each report onward into the two hand-offs the workflow requires: an annotation on the trace in your backend and a reproducer fixture in your local `fixtures/` directory (the shape `mod-106` will eventually consume).

The goal is to internalise the workflow so that on the day someone hands you a trace id and a complaint, the reflex is *step 1 → step 6*, not "let me run it again in staging."

## Prerequisites

- Chapters 04 and 05 of this module.
- The schema from exercise-03 (or chapter 04's support-agent baseline if you did not do exercise-03).
- One of the two chosen backends from exercise-02 running with traces from your instrumented service.
- The ability to attach an annotation / label / score to a trace on that backend (consult the vendor docs).

## Choose an agent

You need a **nested** agent — at minimum, a coordinator agent that calls one sub-agent or two-plus tool calls where the second call depends on the first's output. Options:

- **Extend exercise-01's Q&A service** into a two-step planner: the coordinator emits a plan, then two sequential tool calls (`lookup_order` → `create_followup_email`), then an LLM reply.
- **A LangGraph / LlamaIndex / DSPy sample agent** you already run, with OpenInference or vendor-native auto-instrumentation on.
- **A real internal agent** with employer authorisation, redacted before commit.

Whichever you pick, make sure the trace has at least four child spans under the AGENT root: `LLM.plan → TOOL.a → TOOL.b → LLM.reply`, ideally with a GUARDRAIL span at the end.

## Seed three bad runs

Deliberately introduce three failures. Do NOT skip the seeding — the point is to know the ground truth so you can grade your own bisection. Suggested seeds (pick three):

1. **Wrong tool arguments.** Instruct the plan LLM (via a spiked system prompt) to swap two adjacent digits in a numeric id it passes to the first tool. Chapter 05's worked example.
2. **Retrieval missed the answer.** Ask a question whose answer is in a doc you have removed from the retriever's index; observe an ungrounded reply.
3. **Loop that never terminated (bounded).** Give the planner a system prompt that keeps rewriting the same tool call; cap step count at 5.
4. **Over-refusal.** Add a naive guardrail rule that fires on the word "cancel"; ask a benign question containing "cancel my subscription".
5. **Silent tool failure.** Make the second tool return HTTP 500 with a small error payload; observe how (or whether) the reply surfaces it.
6. **Model version drift.** Pin your request to `gpt-4o-2024-05-13` but have the provider serve `gpt-4o-2024-08-06`; confirm your schema captures both.

Log which seeds you chose. This is your answer key.

## Requirements

Produce a directory `mod-102/exercise-04/` containing:

1. **`bisection-run-<n>.md`** for each of the three seeded runs. Structure exactly matches chapter 05's six steps, in order:
   - **Step 1: Right run?** Quote the root attributes you checked and the one-sentence "this is the right run" statement.
   - **Step 2: Failure signature.** One of the canonical signatures from chapter 05 (or a new one, named).
   - **Step 3: Top-down walk.** The observed child-kind sequence, and any structural anomaly (missing / extra / repeated / wrong-order kind).
   - **Step 4: Descend to suspect span.** Quote the parent's context and the suspect span's `input.value` / `output.value` / relevant attribute. The section is complete when the *smallest span that could have caused the failure* is named and quoted.
   - **Step 5: Bisect by attribute.** Which root attribute would you filter recent traffic by (`experiment.arm`, `service.version`, `gen_ai.response.model`, ...) to test whether the failure class is population-level? Report the actual filter you ran and the result (even if only two comparison traces).
   - **Step 6: Reproduce offline.** The exact fixture JSON in the shape shown in chapter 05, saved next to the report as `fixtures/run-<n>.json`. State whether the fixture reproduced; if not, name what you think the schema is missing.
2. **`annotations.md`** — for each run, the exact annotation text, the label / tag, and (where you attached one) the score you posted to the trace on the backend. Include the trace URL if the backend generates one.
3. **`review-tickets.md`** — for each run, a one-paragraph `needs_triage` ticket you would file into a `mod-109` human-review queue. Structure: title, trace URL, one-sentence hypothesis, one-sentence proposed test, owner.
4. **`missed-signal.md`** — a short reflection listing, per run, at least one attribute you *wished* were on the trace but wasn't. This feeds a schema patch for exercise-03. Empty is unusual; if your first pass didn't produce any, look harder.

## Starter guidance

- Do the seeds one at a time and complete a full six-step report before moving to the next. Do not seed all three, then bisect all three — you will confuse yourself about which is which.
- Quote attributes verbatim from the viewer. Chapter 05's anti-pattern is "the model got confused"; the fix is `llm.tool_calls.0.tool_call.function.arguments = "..."`.
- Time-box each report at ~25 minutes. If step 3 takes longer, your schema is probably the problem — write that finding in `missed-signal.md` and finish the six steps anyway.
- Step 5 is the one people skip. Even a two-trace comparison ("this one has arm A and failed, that one has arm B and did not") is enough to make the section real; it is a hypothesis, not a p-value.
- For step 6, treat "the fixture did not reproduce" as important data. If a trace-based diagnosis cannot be reproduced offline, chapter 05 says the trace did not tell the whole story — that is the signal to enrich the schema.
- Keep annotations short: one paragraph on the backend, not a wall of text. The review-queue ticket is the longer artefact.
- If you are running two backends from exercise-02, do each run in one backend but *cross-check* one of them in the other. Note in `missed-signal.md` whether the other viewer would have led you to the same span faster or slower.

## Acceptance criteria

You are done when:

- Three `bisection-run-<n>.md` files exist, each with all six steps present and each quoting concrete attribute values (not narrative).
- Three `fixtures/run-<n>.json` files exist and can be re-run against the app with a scripted harness (a two-line invocation is enough).
- `annotations.md` lists three annotations that actually appear on the trace on the backend.
- `review-tickets.md` contains three tickets, each with a link that resolves to the trace in your backend.
- `missed-signal.md` names at least one schema gap.
- For every fixture that reproduces the seeded failure, the report explicitly says so; for every fixture that does not, the report names the environmental or missing-schema hypothesis.

## Stretch goals

- Add a **fourth run** from a *real* incident — a trace from your regular traffic that you find surprising, not a seeded one. Bisect the same way. Real runs almost always exercise the schema differently than seeded ones do.
- Fold `missed-signal.md`'s findings back into exercise-03's `SCHEMA.md` as an additive `schema.version` bump. Show the diff.
- Do the same three runs on the *other* backend from exercise-02. Measure the wall-clock delta. If one backend is ≥ 30% faster on this drill for your workload, that is a strong data point for the exercise-02 memo — update it.
- Extend one seeded failure across two services: put the tool behind an HTTP endpoint in a second process, propagate `traceparent`, and confirm the sub-service span still hangs off the coordinator trace under both backends.

## What this exercise does *not* cover

You are not writing the online-eval scorer that would detect this failure class going forward — that is `mod-107`. You are not fully specifying the human-review workflow — that is `mod-109`. Your job here is to prove the schema and the workflow work on your surface. Two hours of practice here saves an afternoon on the day someone reports a bad run.
