# The Sampled-Trace Online-Eval Loop: Sampler, Tier Router, Aggregator, Budget

## Motivation

The picture in chapter 01 has a box labelled "online-eval loop." This chapter opens that box. Everything else in the module — drift monitors (ch. 03), cohort gates (ch. 04), confidence sequences (ch. 05), A/B interface (ch. 06), dashboards (ch. 07) — reads the *output* of the loop. If the sampler is wrong, everything downstream is confidently reading noise.

Three questions the loop must answer, cheaply and continuously:

- **Which of the live traces get scored?** Not all of them — that is not a defensible eval budget on any production surface. Not too few — the drift monitor's window needs enough samples for the p05 to have any meaning. Chapter 02 designs the sampler that hits both constraints.
- **Which judge scores each sampled trace?** The mod-104 tier router assigns OSS / mid / frontier judges *offline* based on rubric criticality and confidence-in-cheap. The same router runs at production frequency here, with the same escalation triggers plus a runtime budget knob.
- **What comes out the other end?** A stream of `{trace_id, cohort_keys, metric, score, judge_tier, cost}` rows landing in an aggregator that computes windowed statistics per cohort and per metric. Everything downstream reads these rows.

This chapter builds those three components — sampler, tier router, aggregator — and derives the arithmetic that keeps the whole thing inside a monthly cap.

## Core concepts

### The loop as four in-order components

```
[production trace stream]
        │
        ▼
   ┌─────────┐   sampled subset
   │ sampler │──────────────────────▶
   └─────────┘
        │
        ▼
   ┌──────────────┐   {trace, tier}
   │ tier router  │─────────────────▶
   └──────────────┘
        │
        ▼
   ┌──────────────┐   {trace, metric, score, cost}
   │ judge runner │─────────────────▶
   └──────────────┘
        │
        ▼
   ┌──────────────┐   {cohort, metric, window_stats}
   │  aggregator  │─────────────────▶  drift monitor, cohort gate,
   └──────────────┘                      conf sequence, dashboards
```

Each arrow is a queue. Each component is stateless except for the sampler's rate-limiter and the aggregator's window buffers. Each queue has a bounded depth and a documented backpressure behaviour (below).

### The sampler: what "1 %" actually means

"Score 1 % of traffic" is under-specified. Four sampling strategies are worth knowing; a real loop composes them.

**Uniform random sampling.** Roll a die per trace; keep it with probability `p`. The default for the baseline stream. Estimator is unbiased; variance follows straight from the sample size. Cheapest to implement.

**Stratified sampling per cohort.** Set a *per-cohort* sample rate — e.g., 0.5 % for the tenant tier that generates 90 % of traffic; 5 % for the tenant tier that generates 5 %; 100 % for the tenant tier that generates 0.5 %. Without stratification, a small-tenant regression is invisible until the on-call notices the aggregate wobble — which never happens because the small tenant is drowned. Rule of thumb: hit at least ~30 – 50 scored traces per cohort per drift window, and derive per-cohort rates from that.

**Deterministic-hash sampling.** Instead of a die per trace, hash a stable id (session id, user id, trace id) and keep the trace iff `hash(id) mod N < K`. Two properties: (a) the same session gets sampled consistently across turns (no half-scored session), and (b) two eval workers scoring the same trace stream will pick the same subset, so you can dedup or shard cleanly. OTel's `TraceIdRatioBased` sampler is this shape.

**Bias-toward-interesting sampling.** Boost the sample rate on traces that carry a signal upstream systems flag — user gave a thumbs-down, tool call errored, low-confidence classifier fired, first-run onboarding tenant, safety filter triggered. This makes the loop *cheaper per surfaced regression* but *biased* — the drift monitor and the confidence sequence must know the sample weight per trace and compute inverse-probability-weighted estimators, or the aggregate is skewed. The bias is worth it; the weighting is required.

Compose them: **stratify per cohort, deterministic-hash within cohort, layer a boost for flagged traces, and record the sample weight on the scored row.** The aggregator does inverse-probability weighting; the drift monitor tolerates the boost; the dashboards show both raw sample counts and weighted estimates.

### The judge-tier router at runtime

Mod-104 chapter 05 built the tier router: cheap OSS judge for the mainline, mid-tier for uncertain cases, frontier for high-stakes ties or safety-critical criteria. At production frequency, three additional runtime knobs matter.

- **Runtime budget guard.** The router has a monthly spend cap. When the running spend is > K % of the cap, downgrade every escalation by one tier (frontier → mid, mid → OSS) and emit a `budget_capped=true` on the scored row. Dashboards must visibly show when the loop is capped; a silent downgrade is how the drift monitor's baseline moves for a reason nobody logged.
- **Rate-limit-aware backoff.** The paid judges have vendor rate limits. The router respects them (429 → exponential backoff + jitter → downgrade if repeated). Traces that could not be scored inside the target window are marked `judge_missed=true` and counted; a `judge_missed` rate above threshold is itself an alert.
- **Judge-tier assignment logged per row.** Every scored row records which tier scored it. When judge-drift (chapter 03) flags a distribution shift on the OSS tier only, the router-level view lets you see whether the shift is a real signal or an artefact of a mid-tier snapshot pin bump.

The routing decision is deterministic given `(rubric_id, mod_104_uncertainty_signal, budget_state)`. That determinism is what makes the tier assignment reproducible when the on-call re-reads a scored row a week later.

### The judge runner: rubric evaluation at runtime

The judge runner is the mod-104 judge, wrapped in an async worker. The wrapper's job is:

- Fan out to `N` concurrent judge calls (typical 8 – 64, tuned to vendor rate limit).
- Batch where the judge API supports it.
- Cache on `(rubric_id, rubric_hash, judge_model_snapshot, trace_content_hash)` so a re-run of the loop on the same trace does not re-pay the judge. Cache TTL is the retention of the trace + judge snapshot lifetime; the cache is *invalidated* on rubric change or judge-model snapshot change.
- Emit `{trace_id, cohort_keys, rubric_id, metric, score, judge_tier, judge_model_snapshot, rubric_hash, cost_usd, latency_ms, sample_weight}` to the aggregator queue.

Every column is load-bearing:

- `judge_model_snapshot` and `rubric_hash` — the reason the row was that score; needed for the chapter 03 judge-drift attribution.
- `cost_usd` — the reason the monthly cap works.
- `sample_weight` — the reason the aggregator's estimator is unbiased.
- `cohort_keys` — the reason the chapter 04 canary gate can slice by variant.

### The aggregator: windowed statistics per cohort and per metric

The aggregator is a streaming rollup. On every scored row it updates:

- **Per-cohort, per-metric sliding window mean, p05, p50, p95, count, sum-of-costs.** Window sizes are configurable — chapter 05 walks the choice; typical defaults are `5min / 1h / 24h` running side-by-side.
- **Per-cohort inverse-probability-weighted mean.** For biased-sampled traces, this is the unbiased estimator.
- **Per-cohort tail-latency percentiles from the trace-timing fields.** Costs and latencies are as ship-blocking as quality (mod-106 chapter 06 asserted this at deploy time; the online loop asserts it at run time).

Two design decisions matter:

- **Windows are computed at read time, not at write time.** Store the raw scored rows; compute the window on query. Chapter 03's drift monitor and chapter 05's confidence sequence want different windows — sometimes wildly different — from the same underlying rows. A pre-aggregated window is an over-fitted store.
- **A per-cohort minimum-sample-size floor.** Below the floor (e.g., `n < 30`), the aggregator returns "insufficient data" instead of a number. A dashboard that shows a p05 computed from 4 samples is a dashboard that gets the on-call paged for noise.

### The budget arithmetic

The whole loop is arithmetic. Derive the number end-to-end for one surface and defend it.

Inputs:

- `R` — requests per second on the surface (e.g., 10 rps).
- `p` — sample rate, aggregate (e.g., 0.01).
- `f_OSS`, `f_mid`, `f_frontier` — fraction of sampled traces routed to each tier (chapter 03 of mod-104's calibration). Sum to 1.0.
- `c_OSS`, `c_mid`, `c_frontier` — $ / call at each tier (e.g., $0.0002, $0.005, $0.05).
- `M` — number of rubrics scored per trace (e.g., 3 for quality / safety / cost).

Derived:

- Sampled traces per day = `R × p × 86 400`.
- Judge calls per day = sampled traces × `M`.
- $ / day = judge calls × `(f_OSS × c_OSS + f_mid × c_mid + f_frontier × c_frontier)`.
- $ / month = $ / day × 30.

Worked example: `R = 10 rps, p = 0.01, M = 3, f_OSS = 0.85, f_mid = 0.10, f_frontier = 0.05, c_OSS = $0.0002, c_mid = $0.005, c_frontier = $0.05`:

- Sampled traces / day = `10 × 0.01 × 86400 = 8 640`.
- Judge calls / day = `8 640 × 3 = 25 920`.
- Mean $ / call = `0.85 × 0.0002 + 0.10 × 0.005 + 0.05 × 0.05 = 0.00317`.
- $ / day = `25 920 × 0.00317 ≈ $82`; $ / month ≈ `$2 460`.

The three levers are `p`, tier fractions, and `M`. When you cannot afford $2 460 / month:

- Cut `M` — score two rubrics per trace, not three; route the third to a nightly batch.
- Cut `p` in the majority cohort; raise it in the minority (stratified sampling, above).
- Push more traces to OSS via a stricter escalation trigger — same as mod-104 but tighter.
- Cap the frontier tier at a fixed daily budget; enforce with the runtime budget guard.

A defensible budget derivation goes in the runbook, in the same document as the threshold. Chapter 06 of mod-106 already sits in the on-call's browser tabs; make the online-loop budget derivation live next to it.

### Backpressure and failure modes

The loop is a pipeline; every stage can be the bottleneck.

- **Sampler → tier router queue full.** Something downstream is slower than the sampler. Do not un-sample (the estimator becomes biased). Drop *before* the tier router; log `dropped_by_sampler_pressure=true`; the dashboards show the drop rate.
- **Tier router → judge runner queue full.** Judge concurrency is at cap. Downgrade escalations (chapter's tier router rule); if that does not clear the queue, drop with `judge_missed=true`.
- **Aggregator behind.** Aggregator lag on a 5-minute window is a real problem for the canary gate. Rate-limit reads to the aggregator; alert on aggregator lag.
- **Judge model outage.** All escalations 429 or 5xx. Every scored row is `judge_missed=true`; alert immediately (this is a monitoring blackout, not a quality regression); the canary gate returns "insufficient data" instead of pass / fail.

Anti-pattern to avoid: **let the sampler blow past the target rate to catch up after a queue drain.** The aggregator will then see a sudden burst of traces from a period that is not the current period; the drift monitor sees a distribution jump; the on-call gets paged. The correct response to a queue drain is to keep the rate constant and mark the interval `partial_coverage=true`.

### The scored-row schema (source of truth)

Every downstream chapter reads this schema. Fix it once; version it.

```json
{
  "scored_at": "2026-08-04T14:32:19Z",
  "trace_id": "5b7e...",
  "session_id": "sess_9c2f...",
  "cohort_keys": {
    "tenant_id": "acme",
    "region": "us-east-1",
    "product_surface": "support_bot",
    "prompt_version": "sha256:abcd...",
    "model_snapshot": "gpt-4o-2024-08-06",
    "retrieval_index_hash": "sha256:1234...",
    "flag_variant": "canary_v3",
    "ab_experiment_arm": "treatment"
  },
  "rubric_id": "faithfulness_v3",
  "rubric_hash": "sha256:beef...",
  "metric": "faithfulness",
  "score": 0.87,
  "judge_tier": "mid",
  "judge_model_snapshot": "claude-3-5-sonnet-20241022",
  "judge_prompt_hash": "sha256:cafe...",
  "cost_usd": 0.0042,
  "latency_ms": 812,
  "sample_weight": 100.0,
  "sampler_stratum": "tenant_tier_large",
  "boost_reason": null,
  "budget_capped": false,
  "judge_missed": false,
  "app_latency_e2e_ms": 1837,
  "app_cost_usd": 0.019
}
```

Read every column: each is required by at least one downstream chapter. Missing `sample_weight` breaks chapter 05's estimator. Missing `judge_model_snapshot` breaks chapter 03's judge-drift attribution. Missing `cohort_keys` breaks chapter 04's canary gate and chapter 06's A/B slice. Missing `app_cost_usd` breaks the mod-111 cost-latency roll-ups. The schema is boring but load-bearing; treat it that way.

### Where this fits the trace backend

The scored-row store is a table in the same warehouse that mod-102's trace backend already writes to (Phoenix, Langfuse, Weave, Braintrust, or a downstream data warehouse). Vendor patterns:

- **Phoenix / OpenInference** — annotations attached to spans; a "score" annotation with attributes matching the schema above.
- **Langfuse** — scores as first-class objects on traces, with metadata for the cohort keys.
- **Weave** — `weave.op` scorers attached to traces; scored objects retrieved via `weave.query`.
- **Braintrust** — online scoring writes to an experiment or a project's dataset; `braintrust.online.score` is the runtime hook.

Pick the vendor whose scoring surface is closest to your existing trace store. The schema is portable; the runtime hook is not. Chapter 07 shows the dashboards these vendors ship out of the box.

## Summary

- The online-eval loop is **four in-order components**: sampler, tier router, judge runner, aggregator. Every arrow is a queue with bounded depth and documented backpressure.
- The **sampler** composes uniform-random, stratified-per-cohort, deterministic-hash, and bias-toward-interesting strategies; the aggregator does **inverse-probability weighting** to unbias biased sampling.
- The **tier router** reuses mod-104's escalation logic plus three runtime knobs — budget guard, rate-limit backoff, and per-row tier logging.
- The **aggregator** stores raw scored rows and computes windows at read time; enforces a per-cohort minimum-sample-size floor.
- The **budget arithmetic** is `R × p × 86400 × M × mean-tier-cost × 30`. Defend the number in the same runbook as the threshold.
- **Backpressure**: drop *before* the tier router; downgrade escalations at queue pressure; never un-sample to catch up.
- The **scored-row schema** is the interface every downstream chapter reads. Fix it once; version it. Missing columns break downstream chapters silently.

Chapter 03 uses the scored-row stream to detect drift at three levels — the input distribution shifting, the output distribution shifting, and the judge itself scoring differently.
