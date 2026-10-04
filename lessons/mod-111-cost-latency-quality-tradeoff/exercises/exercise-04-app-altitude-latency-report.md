# exercise-04: App-Altitude Latency Report

**Estimated effort:** 2 hours

## Objective

Instrument and publish the **app-altitude latency report** from chapter 05: TTFT, TPOT, streaming p50 / p95, retry latency — measured at the request-root span the mod-102 instrumentation emits, sliced per cohort, with a reconciliation footer that attributes the delta against the model-altitude benchmark (vendor console p95, or the `ai-infra-performance` peer's MLPerf-shaped number if one exists).

By the end of the exercise you will have an attribute validator that gates the report on sufficient instrumentation population, a report generator that reads from the mod-102 trace store and emits the four latency metrics per cohort, a reconciliation footer that names the fleet-queue / network / retry attribution of the gap vs. the model-altitude number, and a drill-down for a worst-cohort regression that the on-call rotation can hand to the fleet peer as the chapter 07 escalation ticket.

## Prerequisites

- Chapters 01, 02, and 05 of this module.
- Mod-102 chapter 02 OTel-GenAI trace instrumentation live on at least one product surface, with the attributes chapter 05 lists populated at ≥ 99 % on production traffic. If any load-bearing attribute is below 99 %, fix the instrumentation first — this exercise cannot produce a defensible report against under-instrumented spans.
- A trace backend (Phoenix, Langfuse, Weave, Braintrust, SigNoz, or raw OTel + ClickHouse / DuckDB) reachable from the exercise code.
- A rough baseline for the model-altitude number — either (a) the vendor's published / console p95 TTFT, or (b) the `ai-infra-performance` peer's most recent MLPerf-Inference-shaped benchmark on the org's hardware. If neither exists, the report footer says so and the fallback is "vendor claim, unverified."
- At least 7 days of production spans across ≥ 4 cohorts (or a replayable fixture bundle with at least that distribution).
- Python 3.11+; a histogram / percentile library (`numpy` + `numpy.percentile` is enough); a plotting library.

## Set-up

1. Create `eval/tradeoff/latency/` in your repo:

   ```
   eval/tradeoff/latency/
   ├── config.yaml
   ├── schema/
   │   └── required_attributes.yaml     # the chapter 05 required attribute set
   ├── validate/
   │   ├── populate_rate.py             # population rate per required attribute, per cohort
   │   └── alert.yaml                   # alert config when any attribute drops below threshold
   ├── compute/
   │   ├── ttft.py                      # per-trace TTFT from the stream-start event
   │   ├── tpot.py                      # per-trace TPOT = (end - first-token) / output_tokens
   │   ├── streaming_p95.py             # end-to-end wall-clock
   │   ├── retry.py                     # retry rate, retry-conditional latency, retry-attributed p95 shift
   │   └── cohort_weighted_p95.py       # chapter 05 global vs. cohort-weighted vs. worst-cohort
   ├── reconcile/
   │   ├── vendor_console.py            # pull vendor-reported p95 if available
   │   ├── mlperf_peer.py               # read the peer's benchmark artefact
   │   └── attribute_delta.py           # fleet queue + network + retry + prompt-build attribution
   ├── report/
   │   ├── build_latency_report.py
   │   ├── streaming_shape_plot.py      # TTFT + TPOT histogram per cohort
   │   └── reconciliation_footer.py
   ├── output/
   │   └── <report_id>/
   │       ├── latency_report.md
   │       ├── per_cohort_percentiles.csv
   │       ├── streaming_shape.svg
   │       └── report.json
   ├── tests/
   │   ├── test_ttft.py
   │   ├── test_tpot.py
   │   ├── test_cohort_weighted_p95.py
   │   ├── test_retry.py
   │   ├── test_populate_rate.py
   │   └── test_report_render.py
   └── README.md
   ```

2. Fill `schema/required_attributes.yaml` with the chapter 05 attribute set — the generator refuses to run if any is below the configured population threshold:

   ```yaml
   # chapter 05 required attributes
   model_call_span:
     - gen_ai.system
     - gen_ai.request.model
     - gen_ai.response.model
     - gen_ai.usage.input_tokens
     - gen_ai.usage.output_tokens
     - gen_ai.response.finish_reason
     - first_token_timestamp     # populated as an event or attribute; chapter 05 fallback allowed
   request_root_span:
     - feature.name
     - cohort.locale
     - cohort.tenant_tier
     - retry.attempt
   population_threshold: 0.99
   fallback_rules:
     first_token_timestamp:
       fallback_to: http.first_byte_timestamp
       note_in_footer: true
   ```

3. Fill `config.yaml`:

   ```yaml
   trace_backend:
     kind: phoenix                       # or langfuse | weave | braintrust | otel_clickhouse
     dsn:  http://localhost:6006
   window:
     start: 2026-10-01T00:00:00Z
     end:   2026-10-07T23:59:59Z
   cohorts:
     path: ../routing/cohorts.yaml        # reuse the exercise-02 roster
   percentiles: [50, 95, 99]
   reconciliation:
     vendor_console:
       source:   https://status.anthropic.com      # replace with the vendor your model targets
       baseline: 320                                # ms, p95 TTFT; vendor-reported
     mlperf_peer:
       artefact: ../../../peers/ai_infra_performance/mlperf_latest.json  # or null
   streaming:
     policy: streaming                   # streaming | non-streaming | pseudo-streaming
     stall_threshold_ms: 500
   ```

4. If the `ai-infra-performance` peer publishes an MLPerf-Inference-shaped benchmark, drop its latest artefact into `peers/ai_infra_performance/mlperf_latest.json` so the reconciliation footer can read it. If not, set `mlperf_peer.artefact: null`; the report footer records the absence and uses the vendor console as the sole reconciliation line.

## Requirements

Produce a PR against your working branch that adds:

1. **`schema/required_attributes.yaml`** — the chapter 05 attribute set, with the population threshold and the first-token-event fallback rule.
2. **`validate/populate_rate.py`** — walks a configurable sampling of the production spans in the window and reports population rate per attribute, per cohort. Writes a `populate_rate.csv`; refuses to proceed if any load-bearing attribute is below the threshold on any cohort. Emits a clear error naming the attribute, the cohort, the observed rate, and the suggested instrumentation fix.
3. **`compute/ttft.py`** — per-trace TTFT = `first_token_timestamp − request_root_span_start`. Falls back to `http.first_byte_timestamp` per the fallback rule (and the fallback is noted in the report footer). Returns a per-trace series the percentile code consumes.
4. **`compute/tpot.py`** — per-trace TPOT = `(stream_end − first_token_timestamp) / output_tokens`. Handles the edge case `output_tokens = 0` by excluding the trace (and reporting the exclusion count).
5. **`compute/streaming_p95.py`** — per-trace end-to-end streaming wall-clock = `request_root_span.end − request_root_span.start`. The report shows p50 / p95 / p99.
6. **`compute/retry.py`** — three sub-metrics per cohort:
   - `retry_rate` — fraction of traces with any `retry.attempt > 0`.
   - `retry_conditional_latency` — among traces with `retry.attempt > 0`, the additional wall-clock the retry contributed (first-attempt timeout + retry latency).
   - `retry_attributed_p95_shift` — `streaming_p95(all)` − `streaming_p95(retry.attempt == 0 only)`. Positive values mean retries drive the p95.
7. **`compute/cohort_weighted_p95.py`** — chapter 05's three variants:
   - `global_p95` — all traces together.
   - `cohort_weighted_p95` — per-cohort p95 averaged with configured weights (equal, revenue-share, or declared in config).
   - `worst_cohort_p95` — the max per-cohort p95.
8. **`reconcile/vendor_console.py`** and **`reconcile/mlperf_peer.py`** — load the vendor's reported p95 and the peer's MLPerf artefact (or declare absence). The reconciliation code treats both as inputs; neither is derived here.
9. **`reconcile/attribute_delta.py`** — compute `app_altitude_p95_TTFT − model_altitude_p95_TTFT` and attribute it to:
   - **Fleet queue** — from `fleet.queue.duration` span attribute (if the fleet peer publishes it) or declared as "unknown — escalate to fleet peer."
   - **Network overhead** — from the HTTP-level span's `http.request.duration` minus the inner model-call span duration.
   - **Retry overhead** — `retry_rate × retry_conditional_latency` across the cohort.
   - **Prompt-build overhead** — from the pre-model-call sibling spans (retrieval, chain assembly).
   - **Residual** — the unexplained remainder; a large residual is itself a report finding.
10. **`report/build_latency_report.py`** — the chapter 05 shape:
    - Header block.
    - **Four load-bearing metrics table** — TTFT, TPOT, streaming, retry — p50 / p95 / p99 (`mean` banned), per cohort.
    - **Global / cohort-weighted / worst-cohort p95** — three labelled rows.
    - **Streaming shape** — TTFT + TPOT histogram per cohort; stall count where applicable.
    - **Retry sub-metrics** — rate, conditional latency, attributed p95 shift per cohort.
    - **Reconciliation footer** — vendor / MLPerf-peer / app-altitude; attribution of the delta line by line; residual called out.
    - **Streaming policy** footnote.
    - Report hash in the footer; `report.json` as the machine-readable sibling.
11. **`report/streaming_shape_plot.py`** — a joint TTFT × TPOT histogram per cohort; a stall-count overlay for cohorts with `stall_threshold_ms` exceeded; SVG rendered to the output directory.
12. **A drill-down section for the worst cohort** — in the report, name the cohort with the worst p95, show its percentile distribution, call out whether the gap is TTFT-driven or TPOT-driven, show its reconciliation attribution, and (if the fleet-queue or network attribution is dominant) include a draft of the chapter 07 escalation ticket to `ai-infra-performance` with the cohort data and trace samples attached.
13. **`tests/*`** — unit tests for:
    - `test_ttft.py` — fixture with a known `first_token_timestamp` and `request_root_span_start`; TTFT matches.
    - `test_tpot.py` — fixture with known output_tokens and a known stream duration; TPOT matches; `output_tokens = 0` is excluded with a logged reason.
    - `test_cohort_weighted_p95.py` — a fixture with known per-cohort p95s and declared weights; the weighted p95 matches hand-computed.
    - `test_retry.py` — fixtures with and without retries; rate, conditional latency, and attributed p95 shift match hand-computed values.
    - `test_populate_rate.py` — fixtures with 97 % and 99.5 % population; validator rejects the 97 % and accepts the 99.5 %.
    - `test_report_render.py` — stable Markdown render; the reconciliation footer attribution sums (within rounding) to the gap.
14. **A demonstration run** on ≥ 7 days of real traces (or a representative replay bundle). The report must show:
    - A non-trivial gap between global p95 and worst-cohort p95 — ideally ≥ 20 %.
    - At least one cohort whose latency is TPOT-dominated and one whose latency is TTFT-dominated.
    - A non-zero retry-attributed p95 shift on at least one cohort.
    - A reconciliation attribution where no single row carries 100 % of the delta — fleet queue + network + retry + prompt-build add up; the residual is called out.
15. **`README.md`** — how to validate instrumentation first, how to configure the window and cohorts, how to generate the report, how to read the reconciliation footer, how to draft the escalation ticket from the worst-cohort drill-down. A page walking through the demonstration run.

## Starter guidance

- **Validate instrumentation before you compute anything.** The chapter 05 "static instrumentation" anti-pattern bites hard: a `feature.name` population of 92 % on an older cohort silently drops that cohort from the report entirely, and the on-call rotation only finds out when the enterprise-tier regression goes unreported. The validator is the first job of the report; it must refuse to proceed on under-population.
- **Mean is banned in the latency column.** Chapter 05 is explicit. Every table displays p50 / p95 / p99 — no `mean`. Enforce in the renderer: a `mean` call anywhere in the report generation path is a test-caught failure.
- **Streaming is a shape.** TTFT alone is a scalar; TPOT alone is a scalar; the shape — TTFT distribution next to TPOT distribution next to the stall count — is what the UX report shows. Draw the joint histogram.
- **The reconciliation footer is the single most-skipped section.** Chapter 05's rule: without it, "the app is slow" cannot be attributed and the escalation goes to the wrong peer. Build the attribution first (even if the fleet-queue number is "unknown — escalate" on day one); the number can be filled in later.
- **The residual is a finding, not an embarrassment.** A 60 ms residual that cannot be attributed is a hint the trace instrumentation is missing a span (perhaps the TLS handshake, perhaps a prompt-build sub-step). Call it out in the report; the next iteration shrinks it.
- **Global p95 is for the vendor's dashboard; cohort-weighted p95 is for the SLO.** The report shows both; the SLO is set against the cohort-weighted or worst-cohort variant. When product asks "but the vendor dashboard says 320 ms," the answer is "yes, the vendor's global p95 is 320 ms; our worst-cohort p95 for the enterprise tier is 1.1 s — here's the attribution."
- **Retries are hidden latency unless you look.** Low mean, bimodal distribution, high p99 — classic retry-driven shape. Always report the three retry sub-metrics; cohorts with retry rate above 2 – 3 % are an upstream-degradation signal before they are a latency signal.
- **One drill-down per report, not six.** The exercise asks for a single worst-cohort drill-down because that is the shape of an actionable report. Six drill-downs is a dashboard; one drill-down with the escalation ticket draft attached is an artefact the on-call rotation uses.
- **Avoid tuning the report to the chart you want.** Pick the cohort weights and SLO targets *before* you see the numbers (chapter 02's pre-registration discipline). Switching weights after seeing a worst-cohort make the global p95 look better is motivated reasoning; the report footer records the weighting choice and when it was declared.

## Acceptance criteria

You are done when:

- The validator runs first and refuses the report when any load-bearing attribute is below the declared threshold on any cohort.
- The four load-bearing metrics (TTFT, TPOT, streaming, retry) are present as p50 / p95 / p99 per cohort; no `mean` appears in the latency column.
- Global p95, cohort-weighted p95, and worst-cohort p95 are shown as three labelled rows.
- The streaming shape plot renders per cohort with the TTFT distribution, TPOT distribution, and stall count (where applicable).
- The retry sub-metrics (rate, conditional latency, attributed p95 shift) are present per cohort.
- The reconciliation footer names the model-altitude baseline (vendor console or MLPerf peer), the app-altitude number, the per-attribution-source delta, and the residual.
- A worst-cohort drill-down is present; if the dominant attribution is fleet-queue or network, the drill-down includes a draft chapter 07 escalation ticket.
- Tests pass in CI; `test_populate_rate.py` includes a below-threshold fixture and asserts the generator refuses.
- The `report.json` is reproducible — running the generator twice on the same inputs returns identical bytes.
- The `README.md` walkthrough is followable end-to-end by a reader who has not read chapter 05.

## Stretch goals

- **Live SLO integration.** Export the cohort-weighted p95 as a Prometheus / OpenMetrics series and wire it to the SLO burn-rate alerting rules from mod-110 chapter 06. The report's rollback trigger now has a live signal.
- **Model-vs-fleet-vs-app regression attribution over time.** Compute the per-cohort p95 for every week in the past 90 days; plot the three altitudes side by side. When the app-altitude number drifts up while the model-altitude number is flat, the attribution is fleet-side; when both drift, the model is the source. Automates the chapter 05 reconciliation over time.
- **Streaming stall taxonomy.** For traces with stall count > 0, classify the stall cause by its sibling spans — tool-call wait, retrieval wait, vendor-side pause. The report shows a stall-cause breakdown per cohort.
- **MLPerf-shape translator.** If the fleet peer runs MLPerf but publishes in a format the report cannot read, write a converter that lifts the peer's offline / server scenario numbers into the reconciliation footer's row. The converter is a one-off; the shape is reusable the next time the peer re-benchmarks.
- **End-user latency survey correlation.** Correlate the report's worst-cohort p95 against user-reported "feels slow" rates from the product's feedback instrumentation. When they diverge (app p95 fine, user complaints up), the hint is a UX gap elsewhere (client-side rendering, chunked response buffering).
- **Automatic rollback trigger wiring.** Based on the pre-registered rollback-trigger rule from exercise-01's `rollback/trigger.yaml`, listen to the live p95 series and fire the flag flip automatically. Exercise-01 declared the rule; this stretch wires it.
- **Streaming-vs-non-streaming A/B.** For surfaces where the product is on the fence about streaming, replay traffic under both modes and compare TTFT, TPOT, perceived-wait-time (a stand-in survey metric), and cost. The report decides when streaming earns its complexity.

## What this exercise does *not* cover

You are not building the multi-objective report shape (exercise 01), the routing eval (exercise 02), the replacement-regression harness (exercise 03), or the per-feature cost accounting (exercise 05). You are shipping the *app-altitude latency report with reconciliation* — the chapter 05 artefact that gives the trade-off report's latency column its credibility against the vendor's single number.
