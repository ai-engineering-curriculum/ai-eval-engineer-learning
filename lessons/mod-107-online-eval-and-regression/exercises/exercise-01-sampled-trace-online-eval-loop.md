# exercise-01: Sampled-Trace Online-Eval Loop

**Estimated effort:** 3 hours

## Objective

Stand up the **four-component online-eval loop** — sampler, tier router, judge runner, aggregator — from chapter 02 against the mod-102 instrumented backend. By the end you will have a service that continuously reads live traces, samples them with a defensible strategy, scores the sample with the mod-104 tier router, and writes the chapter 02 scored-row schema to a store the rest of the module reads.

This exercise builds the *plumbing* of the loop. Exercise-02 layers drift alerts on top; exercise-03 layers the canary gate on top; exercise-04 layers the confidence sequence; exercise-05 exports metrics to the A/B platform; exercise-06 builds the dashboards. Get this exercise right; everything else hangs off the scored-row store it produces.

## Prerequisites

- Chapters 01 and 02 of this module.
- The instrumented backend from mod-102 exercise-01 (OTel-GenAI-conforming spans landing in Phoenix / Langfuse / Weave / Braintrust). If the mod-102 exercises are not done, a minimal `answer(question)` service that emits spans is enough.
- Rubrics and tier-router configuration from mod-104 exercise-05.
- A judge API key with a *scoped monthly cap* — `EVAL_ONLINE_KEY` distinct from the production key and the `EVAL_OPENAI_KEY` from mod-106 exercise-01. $50 / month is enough for this exercise; $200 / month is a comfortable ceiling for iteration.
- Python 3.11+; a job runner (a container running as a systemd service, a Kubernetes Deployment, a lambda-with-schedule, or an `asyncio.Task` running under `uvicorn` — pick what your environment supports).
- A relational or column store for the scored rows — Postgres, DuckDB, ClickHouse, or the vendor's native scored-annotation table.

## Set-up

1. Create `eval/online/` in your repo:

   ```
   eval/online/
   ├── config.yaml
   ├── sampler.py
   ├── tier_router.py
   ├── judge_runner.py
   ├── aggregator.py
   ├── schema.py
   ├── main.py
   ├── budget.md
   └── runbooks/
       ├── judge_missed.md
       └── budget_cap.md
   ```

2. In `config.yaml`, declare the surface, the target sample rate, the per-cohort strata, the tier fractions, the monthly cap, and the rubric ids to score. Example skeleton:

   ```yaml
   surface: support_bot
   sampler:
     strategy: hash          # deterministic-hash on session_id
     aggregate_rate: 0.01
     strata:
       - key: tenant_tier
         rates: {enterprise: 0.05, growth: 0.02, self_serve: 0.005}
     boost:
       - reason: user_thumbs_down
         multiplier: 20.0
       - reason: safety_filter_triggered
         multiplier: 50.0
   tier_router:
     default_tier: oss
     escalate_uncertain: mid
     escalate_safety: frontier
     monthly_cap_usd: 200
     downgrade_at_percent: 80
   rubrics: [faithfulness_v3, safety_refusal_v2, groundedness_v3]
   ```

3. In `schema.py`, define a typed `ScoredRow` dataclass matching the chapter 02 schema verbatim. Enforce every column present (raise on missing `sample_weight`, `judge_model_snapshot`, `rubric_hash`).
4. Provision the store — a Postgres table or a ClickHouse table (the vendor's own scored-annotation table is also fine, so long as every schema field is preserved).
5. Author `budget.md` — the derivation of your `p`, tier fractions, and monthly spend, per chapter 02's arithmetic. This is a real deliverable; a runbook whose budget derivation lives in someone's head is not defensible.

## Requirements

Produce a PR against your working branch that adds:

1. **`sampler.py`** — implements uniform, stratified, deterministic-hash, and bias-toward-interesting sampling. Composes them per chapter 02. Records `sample_weight`, `sampler_stratum`, and `boost_reason` on every emitted row. `sample_weight` correctly reflects inverse selection probability. Unit tests confirm the weighted-mean estimator is unbiased against a simulated stream.
2. **`tier_router.py`** — reuses mod-104's escalation logic; adds the three runtime knobs (budget guard, rate-limit backoff, per-row tier logging). Records `judge_tier` and `budget_capped` on the row. Unit tests confirm downgrade behaviour at ≥ `downgrade_at_percent` of the monthly cap and correct exponential backoff on 429.
3. **`judge_runner.py`** — async fan-out (concurrency configurable), on-disk cache keyed on `(rubric_id, rubric_hash, judge_model_snapshot, trace_content_hash)`, emits the full scored-row schema. Records `cost_usd`, `latency_ms`, `judge_missed`. Confirm cache hit-rate on a re-run against the same trace window.
4. **`aggregator.py`** — a streaming rollup with per-cohort sliding-window statistics (5-minute / 1-hour / 24-hour running side by side). Enforces the per-cohort minimum-sample-size floor (return `INSUFFICIENT_DATA`, not a number, below the floor). Computes both raw and inverse-probability-weighted means.
5. **`main.py`** — the loop entrypoint. Reads config, opens a stream from the trace backend (per-vendor SDK — Phoenix's `Client`, Langfuse's `fetch_traces`, Weave's `query`, Braintrust's `experiments.fetch`), passes traces through the four components, writes scored rows to the store.
6. **`runbooks/judge_missed.md`** — chapter 02's `judge_missed=true` alert runbook. Owner, threshold, first step (check vendor status page and rate limits), remediation, escalation.
7. **`runbooks/budget_cap.md`** — chapter 02's `budget_capped=true` alert runbook. Owner, first step (visible-on-dashboard confirmation), remediation options (raise cap, tighten tier router, cut sample rate).
8. **`budget.md`** — the end-to-end budget derivation with real numbers for your surface: `R`, `p` per stratum, tier fractions, monthly $ / month with the arithmetic shown. Update when any input changes.
9. **A demonstration run** — run the loop against real traces for at least 1 hour on a live or replayed stream; the scored-row store contains at least 200 rows spanning at least 2 cohorts and 2 rubrics; the aggregator returns non-`INSUFFICIENT_DATA` values on the aggregate. Screenshot or paste the aggregator output.

## Starter guidance

- **Do the deterministic-hash sampler first.** Uniform-random is easier to reason about but gives you split sessions and unstable dedup. Chapter 02's `hash(session_id) mod N < K` is the correct default; the other strategies compose on top.
- **Sample weight is not optional.** The single most common bug in a biased-sampling loop is forgetting to propagate the inverse probability. The aggregator's weighted mean is *how* biased sampling stays unbiased; skip the weight and the drift monitor's numbers are wrong.
- **Cache aggressively.** Judge cost is the dominant line item; a re-run of the loop on the same trace window with the same rubric hash should be near-zero cost. Test this — a rebuild that re-pays the judge budget is a bug.
- **Emit every field on the scored-row schema, even if some are `null`.** Downstream chapters (exercises 02–06) each assume specific fields. A missing field is a silent break; a `null` is a documented absence.
- **Backpressure**: choose "drop-before-sampler" for the sampler-router queue and "downgrade-then-drop" for the router-runner queue, per chapter 02. Log every drop with a reason.
- **Vendor-neutral schema, vendor-native runtime.** The scored-row schema is portable; the trace-fetch SDK is not. Wrap the fetch in an adapter (`TraceSource` abstract with per-vendor implementations); the rest of the code depends on the adapter, not the SDK.
- **Do the budget math before the code.** If the numbers in `budget.md` do not fit inside a defensible monthly cap, the loop's design is wrong and no amount of engineering will make the cap work. Cut `M`, cut `p`, tighten the tier router *at the design step*.
- **Do not skip the runbooks.** `judge_missed` and `budget_capped` are the two alerts your loop will fire in its first month; the on-call needs to know what to do. Runbooks are executable, not aspirational (mod-106 chapter 03 rule).

## Acceptance criteria

You are done when:

- The loop runs continuously against the trace backend for ≥ 1 hour with observable throughput matching the target sample rate (± 20 %). `sampler_stratum` and `sample_weight` are populated on every row.
- The scored-row store contains at least 200 rows spanning ≥ 2 cohorts and ≥ 2 rubrics; the schema is enforced (no `null` in required columns beyond documented ones).
- Cache hit-rate on a re-run of the same trace window is ≥ 95 %; total judge cost of the re-run is < 5 % of the initial run.
- The aggregator returns non-`INSUFFICIENT_DATA` values on the aggregate and correct `INSUFFICIENT_DATA` values on small cohorts below the floor.
- Simulating `judge_missed` (e.g., mock the judge to raise) triggers the runbook link in the alert; simulating `budget_capped` triggers the downgrade behaviour in the tier router.
- `budget.md` reflects the real per-surface derivation; the numbers reconcile against the actual cost accrued during the demonstration run (within 30 %).
- `README.md` in `eval/online/` briefly documents how to start the loop, how to query the store, and how to interpret the aggregator output.

## Stretch goals

- **Bias-toward-interesting on ≥ 2 signals.** Wire `user_thumbs_down` (from the mod-102 feedback trace attribute) and `safety_filter_triggered` (from the input classifier) as boost reasons; confirm the inverse-probability-weighted mean stays within 5 % of the uniform-only mean on the same data.
- **Vendor swap.** Wrap two `TraceSource` implementations (e.g., Phoenix and Langfuse). Confirm the same downstream code produces the same aggregator output on the same underlying data. This is what makes the loop portable when the vendor changes.
- **Async concurrency tuning.** Run the loop at 8 / 16 / 32 concurrent judge calls; measure throughput, tail latency, and rate-limit-triggered downgrades. Pick the concurrency that maximises throughput at ≤ 1 % rate-limit downgrades and document the choice.
- **Judge cache invalidation on rubric bump.** Ship a small "rubric bumped" workflow: change the rubric text, confirm `rubric_hash` changes, confirm the cache is not read on the changed rubric, confirm the old cache entries are eligible for pruning after retention window.
- **OTel-native score annotation.** Emit the scored rows *back* into the trace backend as OTel span annotations (following Phoenix's `openinference.instrumentation.annotate` shape or the equivalent), in addition to the store. Confirm the score is visible on the mod-102 trace explorer.
- **Backpressure telemetry.** Instrument every queue with depth / drop-rate metrics; wire them to the eval-team loop-health dashboard skeleton for exercise-06.

## What this exercise does *not* cover

You are not building the drift monitor (exercise-02), the canary gate script (exercise-03), the confidence-sequence library (exercise-04), the A/B interface (exercise-05), or the dashboards (exercise-06). You are shipping the *scored-row producer* the next five exercises consume.
