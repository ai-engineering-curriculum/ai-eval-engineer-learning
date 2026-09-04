# App-Altitude Latency: The Way Users Experience It

## Motivation

The vendor's console shows a p50 latency of 480 ms for the frontier model. The app's p95 streaming latency is 4.1 seconds. Both numbers are true. Only one of them is the number the user experiences.

The gap between the two is where most latency-regression escalations start. A candidate model looks great on the vendor's benchmark; the eval team ships it; users complain about slowness; nobody can reconcile the vendor number with the user experience because the vendor's number is a single-request, no-tool-call, no-retry, no-prompt-build, model-altitude number, and the user experience is an app-altitude number that includes every one of those overheads.

This chapter builds the app-altitude latency report the trade-off surface (chapter 02) needs. Four load-bearing metrics — **TTFT, TPOT, streaming p50 / p95, retry latency** — measured at the request-root span from the mod-102 instrumentation, sliced by cohort, with a reconciliation footer that names the delta against the model-altitude MLPerf-shaped benchmark and attributes it. If the peer `ai-infra-performance` owns a serving benchmark, the report reads it too and closes the loop.

## Core concepts

### App altitude vs. model altitude vs. fleet altitude

Chapter 01 introduced the three altitudes; latency is the axis where the split is sharpest.

- **Model-altitude latency.** One request, one model, one hardware target, no chain. The vendor's console number. MLPerf Inference measures this — Server / Offline scenarios; batched throughput; single-stream latency. Owned by `model-evaluation-engineer` at level 30. The measurement rig is idealised: no prompt build, no retrieval, no tool call, no retry, cold-start warmed away, network to the endpoint negligible.
- **Fleet-altitude latency.** The model plus the serving stack on the org's hardware. Kernel choice, batching policy, KV-cache eviction, autoscaler headroom. `ai-infra-performance`'s number. Adds queue time, batching overhead, cold-start on scale-out, cross-AZ round-trip. Not the user's number, but closer.
- **App-altitude latency.** The full user-visible request cycle. Prompt build (retrieval, tool selection, chain assembly), the model call (which may include queue + batch + inference at the fleet altitude), tool-call round-trips (each of which may recurse into another model call), retries on transient errors, streaming decode buffering, client-side rendering. This module's number. The one the user experiences.

The app-altitude number is the *sum-of-parts* — never a single call. A typical support-bot request:

```
+-- (1) prompt build .......................... 90 ms
|     +-- retrieval span .............. 65 ms
|     +-- prompt assembly .............. 25 ms
+-- (2) first model call ..................... 620 ms  (TTFT)
|     +-- fleet queue ................. 40 ms
|     +-- vendor round-trip ............ 580 ms  (model-altitude p95 320 ms)
+-- (3) tool call [search] ................... 340 ms
|     +-- external API ................ 320 ms
|     +-- span-write flush ............. 20 ms
+-- (4) second model call .................... 380 ms  (TTFT of continuation)
+-- (5) streaming decode ..................... 2400 ms  (TPOT p50 22 ms, ~110 tokens)
+-- (6) client render buffer ................. 180 ms
                                            = 4010 ms  end-to-end streaming p95
```

Every layer is a span from mod-102. The report attributes the p95 to specific layers; the eval team can then negotiate with the fleet peer, the vendor, and the product team over which layer to shrink.

### The four load-bearing metrics

Four metrics carry the latency story. Every one is a p50 / p95 / p99 — never a mean.

- **TTFT — Time To First Token.** From the request-root span start to the first token event on the model call's streaming span. What "the model started responding" feels like to the user. The single most-perceived latency metric.
- **TPOT — Time Per Output Token.** The average inter-token gap on the streaming call. What "the model is thinking" feels like once the response starts. Product-visible on long responses; hidden on short ones.
- **Streaming p50 / p95 — end-to-end wall-clock of the request-root span.** TTFT + (output-tokens × TPOT) + everything else (tool calls, retries, prompt build). The whole-experience number.
- **Retry latency.** The additional latency introduced when the first attempt failed (5xx, timeout, tool-call error) and the retry succeeded. Distinct from streaming p95 because most retries do not fire; when they do, they add hundreds of ms to seconds. Track the retry *rate* and the retry-conditional latency separately.

Two secondary metrics the report also carries:

- **Tool-call round-trip median.** The median wall-clock of a tool-call sub-span. Escalates when an external API degrades; visible on the report before the user reports it.
- **Hard-timeout rate.** The fraction of requests that hit a hard timeout (typically 30 s or 60 s at the app boundary). Every hard timeout is a user-visible failure; this rate is a first-class metric.

Every metric is sliced by cohort. Some cohorts (long-context, tool-heavy, enterprise with strict policy checks) have systematically different tail latencies; the average across cohorts hides the tail.

### OpenTelemetry GenAI attributes: the measurement contract

Chapter 02 of mod-102 wired the trace instrumentation to the OTel-GenAI semantic conventions. The latency report reads from those spans; the exercise for this chapter checks the attributes exist and are populated.

Load-bearing attributes on the model-call span:

- `gen_ai.system` — vendor identifier (`openai`, `anthropic`, `google.vertex`, `aws.bedrock`, `custom`).
- `gen_ai.request.model` — the requested model id (may be an alias).
- `gen_ai.response.model` — the actual model snapshot served (may differ from the alias if the vendor routed).
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` — the token counts (feed into chapter 06 too).
- `gen_ai.request.temperature`, `gen_ai.request.top_p`, `gen_ai.request.max_tokens` — decoding config.
- A stream-event with the timestamp of the *first token* — this is what TTFT is computed against. The OTel-GenAI spec uses an event `gen_ai.content.completion` (or the semantic-convention-approved analogue for the vendor); when the event is absent, the app-altitude TTFT falls back to the first-byte timestamp on the HTTP-level span and the report footer notes the fallback.
- `gen_ai.response.finish_reason` — populated at the end of the stream; a `length` finish is a candidate for tuning `max_tokens`.

Load-bearing attributes on the *request-root* span (the app boundary):

- `feature.name` — the product feature identifier (`support_bot.answer`, `research.summarise`, `code.completion`). Chapter 06 uses this for per-feature cost attribution; the latency report uses it for feature-scoped p95.
- `cohort.locale`, `cohort.tenant_tier`, `cohort.query_type`, `cohort.input_length_band` — the report's cohort dimensions.
- `retry.attempt` — the attempt number (0 for the first try, 1 for the first retry, ...); enables the retry-conditional latency calculation.

Missing attributes are a common failure mode. The exercise's first step is a validator that walks a day of production spans and reports the population rate of every attribute the report needs. Population below 99 % on any load-bearing attribute means the instrumentation is unfinished; fix mod-102 first.

### Where p95 comes from

The report displays `p95` as if it is one number. In production traffic, p95 is a *distribution over cohorts weighted by traffic share*, and how that weighting is done matters.

Two ways to compute a "p95":

- **Global p95.** Sort every request in the window by latency; take the 95th percentile. The number is weighted by whatever cohorts have the most traffic. A p95 dominated by the `en-US free-tier short-input` cohort tells the on-call rotation almost nothing about the enterprise tenant's experience.
- **Cohort-weighted p95.** For each cohort, compute its own p95; then take a weighted average across cohorts, with a weight that reflects the cohort's importance (revenue share, SLA obligation, or equal weight if all cohorts are peers).

The report shows **both**, labelled distinctly. The global p95 is what the vendor's dashboard shows and what naive comparisons happen to compare against. The cohort-weighted p95 is what the SLO should be set against.

For SLO purposes, the report also names the **worst-cohort p95** as its own metric. A p95 of 3.9 s that includes an enterprise-cohort p95 of 8.2 s is a red flag even if the global p95 looks acceptable.

### Reconciliation with the model-altitude benchmark

Every latency report has a reconciliation footer.

```
## Reconciliation vs. model-altitude benchmark

Vendor benchmark (from vendor console, single-request, no chain):
  frontier-v3 p95 TTFT = 320 ms
  frontier-v3 p95 TPOT = 18 ms

MLPerf Inference — Server scenario (peer report):
  frontier-v3 p99 first-token = 380 ms  (batch=32)

App-altitude p95 TTFT (this report, en-US cohort):
  480 ms  (Δ +160 ms vs. vendor)

Attribution of the delta:
  - fleet queue (from ai-infra-performance): +45 ms   (queue depth in-region)
  - network overhead (client → app → vendor endpoint): +85 ms  (cross-region + TLS)
  - retry overhead (0.3% retry rate, ~1200ms per retry): +30 ms
                                                       ------
                                                       +160 ms
```

The reconciliation is not optional. Without it:

- The vendor's number is used as the app's number and nobody catches the fleet or network overhead.
- The MLPerf-shaped benchmark from the peer looks incomparable to the app number and neither peer nor eval team can share a language.
- Regressions in the app number that come from fleet or network changes are attributed to the model, and the on-call escalation goes to the wrong team.

Reconciliation lets the report say "yes, our number is 160 ms above the vendor's number, and here is exactly why." Product accepts that framing; "we are slow" is not a defendable framing.

The MLPerf-Inference-shaped benchmark is the peer's substrate. The peer produces the benchmark against a fixed dataset (LLaMA-2-70B on the [MLPerf Inference LLM tasks](https://mlcommons.org/benchmarks/inference-datacenter/), for example), on the org's hardware. The app-altitude report reads the peer's latest number and reconciles against it. If the peer does not run MLPerf-shaped benchmarks, this reconciliation row uses the vendor's console p95 as a fallback and the report footer names it.

### The retry-latency contract

Retries are where hidden latency lives. Three sub-metrics:

- **Retry rate.** Fraction of requests that fired at least one retry. Baseline is typically 0.1 – 1 %; a rate above 2 – 3 % is a signal the upstream is degrading.
- **Retry-conditional latency.** For requests that fired a retry, the additional wall-clock the retry contributed. Typically 500 – 2000 ms — the timeout of the failed attempt plus the retry attempt's latency.
- **Retry-attributed p95 shift.** How much of the streaming p95 is attributable to retries. Computed as `(streaming_p95 including retries) − (streaming_p95 restricted to no-retry requests)`. When retries drive p95, the number is large.

The report shows all three, per cohort. A cohort with a normal streaming p95 but a 10 % retry rate is a cohort where either the vendor is degrading for that cohort's specific request pattern, or the app-layer retry policy is over-eager. Both are actionable.

Two retry-policy failure modes worth naming:

- **Retry on non-idempotent errors.** The first attempt actually completed but the response was lost; the retry causes double side-effects (e.g., a tool call fired twice). Every retry-eligible operation must be idempotent, or the retry policy must inspect the specific error code before firing. Chapter 05 of mod-108 covered this from the safety side; the latency report catches the symptom (double latency + spend on double-fires).
- **Fixed-interval retry on the same endpoint.** A pod is degraded; every retry hits the same pod; every retry fails; the request hard-times-out after 3 × timeout. Backoff-with-jitter and per-attempt endpoint rotation are the fix; the report's retry-rate spike is the signal.

### Streaming as a first-class latency shape

Non-streaming LLM responses have one latency number — the time to the full response. Streaming responses have a shape:

- **TTFT.** Time until the user sees the first character.
- **TPOT distribution.** The gaps between tokens. A stream that arrives in bursts (150 ms, then 250 tokens in a flurry, then 300 ms of nothing, then a flurry) feels worse than a stream with steady 40 ms inter-token gaps, even if the final `TTFT + total_output_time` is the same.
- **Stall count.** The number of inter-token gaps above a threshold (e.g., 500 ms). A steady stream with zero stalls is a different UX from one with 3 stalls.

For product surfaces where streaming is central (chat, summarisation, code completion), the report carries at least TTFT p50 / p95 and TPOT p50 / p95 / p99. For surfaces where the response is short and rendering is atomic (JSON tool output, classification), only end-to-end latency matters and TPOT is a footnote.

The report footer names the streaming policy: `streaming`, `non-streaming`, `pseudo-streaming` (accumulate on server, send atomically), or `streaming with typing effect` (rate-limit the client render). Product decisions about streaming (turn it on for chat, off for JSON output) show up in this footer.

### App-altitude latency in the trade-off report

Chapter 02's report shape carries the latency column. What this chapter adds:

- The `latency` column becomes TTFT p95, TPOT p95, streaming p95, retry-rate — four sub-columns, per cohort.
- The reconciliation footer becomes a required section of the report ("Latency reconciliation" with the vendor / MLPerf / app-altitude rows).
- The pre-registered rule can (and should) name specific latency thresholds against specific cohorts: "streaming p95 on the enterprise cohort does not exceed 3.5 s."
- The rollback trigger can reference the retry rate — a spike above 2 % on any cohort auto-fires the rollback.

### Anti-patterns to avoid

**"Vendor p50 is our latency."** Chapter 01's anti-pattern; the reconciliation footer is the discipline.

**"Global p95 only."** The p95 that hides the enterprise-cohort tail. Report both global and cohort-weighted; call out worst-cohort p95 as a first-class metric.

**"Mean latency."** Means are as misleading for latency as for quality — the mean is close to p50; the tail is what users escalate about. Mean is banned in the latency column.

**"Streaming reported as one number."** A stream is a shape. TTFT and TPOT together. A p95 for the full stream without the shape hides UX regressions where TTFT stays flat but TPOT explodes.

**"Retry rate absent."** Retries are hidden latency. When the report does not carry retry rate and retry-conditional latency, an outage upstream shows up as "the app is slow" with no attribution.

**"Latency in isolation from cost."** Some latency reductions cost money (bigger context caches, more headroom, a cheaper-but-faster model with worse quality). The multi-objective report puts them together; the latency report never lands in a decision without the other three columns.

**"Model-altitude benchmark ignored."** The peer's MLPerf-shaped number is there for a reason. Reconcile against it. When the app number is far above the model number, the delta is fleet or app-layer; when the app number is close to the model number, the model itself is the bottleneck. Different conclusions, different owners.

**"Static instrumentation."** The OTel-GenAI attributes drift as vendors release new SDKs and the OTel spec evolves. The instrumentation validator (exercise 04) runs continuously; the report footer names the instrumentation version.

## Summary

- **App altitude** (this module) is the latency the user sees; **model altitude** (`model-evaluation-engineer`) is the single-request vendor / MLPerf number; **fleet altitude** (`ai-infra-performance`) is the serving-stack number in between. The app number is the sum-of-parts; the reconciliation footer attributes the delta.
- The **four load-bearing metrics** are **TTFT**, **TPOT**, **streaming p50 / p95**, **retry latency** — all as p50 / p95 / p99, all per cohort, `mean` banned.
- The **OTel-GenAI attributes** on the model-call span (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`) and the request-root span (`feature.name`, `cohort.*`, `retry.attempt`) are the measurement contract; a validator checks their population rate.
- The report shows **global p95**, **cohort-weighted p95**, and **worst-cohort p95** as three distinct numbers. The SLO is set against the cohort-weighted or worst-cohort variant; the global is where the vendor dashboard drifts.
- **Reconciliation** against the model-altitude benchmark (vendor console or MLPerf Inference from the peer) is required — the delta is attributed to fleet queue, network overhead, retry rate, prompt build, tool calls.
- **Retry latency** is a first-class contract: retry rate, retry-conditional latency, retry-attributed p95 shift. All three per cohort. Retries on non-idempotent operations and fixed-interval retry on the same endpoint are the two failure modes to name.
- **Streaming is a shape**, not a scalar — TTFT p95 and TPOT p95 together; stall count for streaming-central surfaces. The streaming policy is a report footer.
- **Anti-patterns**: vendor p50 as app latency; global p95 only; mean latency; single-number streaming; missing retry rate; latency reported without cost; MLPerf reconciliation skipped; instrumentation validated once and forgotten.

Chapter 06 opens the cost column of the trade-off report — per-feature token accounting with input / output / cache-read / cache-write / batch splits, the pricing snapshot, and how the cost report drives product decisions.
