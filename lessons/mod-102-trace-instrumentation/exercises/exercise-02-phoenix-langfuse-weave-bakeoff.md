# exercise-02: Phoenix / Langfuse / Weave / Braintrust / LangSmith Bake-Off

**Estimated effort:** 3 hours

## Objective

Wire the instrumented service from exercise-01 through an **OpenTelemetry Collector** that fans out to two of the five backends covered in chapter 03, run a scripted debugger drill on both, and write the "pick-a-backend" memo you would hand to your tech lead. The point is to build a defensible choice *on your workload*, not to trust a vendor comparison matrix.

## Prerequisites

- Chapters 02 and 03 of this module.
- Exercise-01 complete — you need at least one of `app_otel_genai.py` / `app_openinference.py` running and posting traces via OTLP.
- Docker (or an equivalent container runtime) to run a local Collector and, if you pick it, a local Phoenix.
- Free-tier / trial accounts for **two** of the five backends: Phoenix (self-host counts), Langfuse, W&B Weave, Braintrust, LangSmith. Read the current docs pages linked in `resources.md` for signup and OTel-ingest instructions — do not trust a friend's URL, they move.
- ~$5 of model spend budget for the drill traffic.

## Choose two backends

Pick pairs deliberately — do not pick two vendors from the same camp. Suggested pairings:

- **Phoenix (self-host) + LangSmith (SaaS).** Convention-native OpenInference viewer vs LangChain-native.
- **Langfuse (self-host) + Braintrust (SaaS).** Session-first vs eval-first data models.
- **Weave (SaaS) + Phoenix (self-host).** W&B-integrated vs standalone LLM-app viewer.

If your organisation already commits to one, use that as one of the two and make an honest second pick — the bake-off's job is to check your default, not confirm it.

## Set-up

1. **Deploy an OTel Collector locally** using the `otel/opentelemetry-collector-contrib` image (chapter 03). Configure it with:
   - One OTLP receiver (`grpc` + `http`).
   - Two `otlphttp` exporters, one per chosen backend. Cross-check the endpoint URL, headers, and auth mechanics against the current vendor docs; do not copy the chapter-03 sketch verbatim.
   - The `batch` processor. Nothing else in the pipeline for this exercise.
2. **Point exercise-01's service at the Collector** (`OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`).
3. **Generate ~200 traces** by running exercise-01's five inputs in a loop, with light per-run variation (different order ids, occasional deliberately-empty question). Include ~5 seeded failures: a tool call with a bad argument, a request that times out, a request that returns a very long response (~4 KB), and one that trips a simple content check.

## Requirements

Produce a directory `mod-102/exercise-02/` containing:

1. **`collector.yaml`** — the working Collector config, with real endpoint URLs (secrets externalised via env). Include a short comment block at the top pointing at the vendor doc pages you followed.
2. **`bakeoff-scorecard.md`** — a table with one row per backend and the following columns, filled in from actual observation, not marketing pages:
   - `Convention leaning` (OpenInference / GenAI / vendor-native, and how the viewer renders your traces).
   - `Trace-tree legibility` (1–5, with a one-sentence justification).
   - `Session model` (first-class / query / absent).
   - `Time to open a specific failing trace from a session id` (median wall-clock over 5 attempts, in seconds).
   - `Time to identify the root-cause span in a seeded failure` (median wall-clock over 3 drills).
   - `Ease of attaching a numeric score to a span` (1–5, with an example call).
   - `Self-host feasibility` (yes / partial / SaaS-only — cite the docs URL you read).
   - `PII / hosting posture` (single sentence — cite the docs URL).
   - `Notable friction` (one line per backend of the thing that surprised you).
3. **`drill-log.md`** — the raw log of the three seeded-failure drills, one section per drill, per backend. Each section records: the seeded failure, the query you filtered by, the click path in the UI, the span attribute name and value that pinned the root cause, and the wall-clock time. This is the input to the scorecard; keep it honest.
4. **`memo.md`** — 500–800 words. Structure:
   - **Recommendation.** One paragraph naming the primary backend for the surface, optionally naming a secondary that keeps for four weeks.
   - **Reasoning.** Two paragraphs, each grounded in the scorecard. Do not use words like "leading" or "industry-standard" — cite observed behaviour or docs pages instead.
   - **What the bake-off *did not* answer.** One paragraph listing at least three questions the drill cannot answer (throughput at 10× load, cost at production volume, feature X on the roadmap, ...) and how you would escalate them (procurement, security review, staged rollout).
   - **Migration cost.** One paragraph estimating how much code would change if the primary pick turned out wrong in six months. If the answer is "much", the memo should say so.

## Starter guidance

- Read each vendor's OpenTelemetry-ingest doc page front-to-back before you touch config. Ingest URL formats and auth headers are the two things most likely to bite; endpoints have moved in past releases.
- Time your drills with a stopwatch, not a memory. The wall-clock times are the least-fungible column in the scorecard.
- On each drill, save the click-path and the exact span attribute you quoted. Chapter 05's bisection loop should feel identical on both backends; if it doesn't, name the difference in `drill-log.md`.
- Do not attempt to render every attribute in every viewer. Some backends render OpenInference natively; others show it as raw key-value. Note the rendering gap; it is often the deciding factor.
- If a backend's docs say a feature "requires an enterprise plan" or is behind a waitlist, mark it clearly and lower its score with a note — do NOT infer capabilities you cannot exercise.
- Keep the Collector process attached to your terminal for the exercise so failures fail loudly. Use `--config=collector.yaml` and `otelcol-contrib validate --config=collector.yaml` before running.

## Acceptance criteria

You are done when:

- `collector.yaml` validates and successfully fans out one live trace to both backends.
- The 200-trace drill traffic is visible in both viewers within a minute of generation.
- All ten scorecard columns are filled for both backends, with the wall-clock times supported by `drill-log.md`.
- `memo.md` names a primary, a reason, an admission, and a migration estimate.
- No secret is committed to the repo (Collector auth headers reference env vars).

## Stretch goals

- Add a third exporter to the same Collector for a **vendor-neutral** sink (Jaeger or Grafana Tempo). Note in the memo how much of the LLM-app-shaped debugging survives when the viewer does not understand the conventions.
- Add the `openinference-instrumentation-openai` (or -anthropic) auto-instrumentor to the app and re-run the drill. Note in `drill-log.md` whether the auto-instrumented spans are legible in both viewers or only in the one that speaks OpenInference natively.
- Have a peer perform the three drills without your help against both viewers. Their wall-clock times replace yours in the scorecard — this is the real "on-call opens the viewer at 2am" test.

## What this exercise does *not* cover

You are not designing the schema (exercise-03) or the sampling policy (exercise-05). You are not committing to a backend for your team's production surface — the memo is a *recommendation*, and the real decision usually involves procurement and security teams the exercise does not model.
