# Sampling Policy and PII Scrubbing at Trace Budget

## Motivation

Full-fidelity tracing does not scale. A chatty LLM surface — support agent, coding assistant, meeting summariser — pushes tens to hundreds of KB of text through every trace: prompt, completion, retrieved chunks, tool payloads. At 100% capture, judge-scoring costs on top (mod-104, mod-107) dominate the infra bill within a quarter, and the storage cost of raw payloads collides with the PII posture your Governance-family peer expects.

The two policies that solve this — **sampling** (what fraction of traces do we keep, and which ones) and **PII scrubbing** (what fraction of a kept trace's *payload* survives) — are the last mile of trace instrumentation. They need to be written down, deployed alongside the app, budgeted against a real dollar cap, and stated clearly enough that a Governance-family peer (peer role `ai-evaluation-engineer`, level 25) can sign off. This chapter walks the three sampling positions, the two-layer scrubbing pattern, and how to hold the SLIs from mod-101 legible under both.

## Core concepts

### Head, tail, and signal-driven sampling

There are three positions in the pipeline where a keep-or-drop decision can happen. Each has a role.

**Head sampling** — the decision is made at span-creation time, in the SDK (or a parent-based sampler that inherits from an upstream service). Fast, cheap, no buffering. Available primitives in the OpenTelemetry SDK are the `ParentBased`, `TraceIdRatioBased`, `AlwaysOn`, and `AlwaysOff` samplers (<https://opentelemetry.io/docs/specs/otel/trace/sdk/#sampling>). What head sampling *cannot* do: keep a trace because it turned out to be interesting. If you head-sample at 5% and the interesting failure was the 6th trace, it is gone.

**Tail sampling** — the decision is made at the OpenTelemetry Collector, after all spans of a trace have arrived and been buffered for a window. The `tail_sampling` processor in the `opentelemetry-collector-contrib` distribution (<https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor>) supports policies for `status_code`, `latency`, `numeric_attribute`, `string_attribute`, `rate_limiting`, `trace_state`, `and`, `composite`, and `ottl_condition`, among others. It costs memory and adds decision latency, but it lets you keep 100% of errors, high-cost traces, and canary-arm traces without keeping 100% of everything.

**Signal-driven sampling** — the keep-or-drop decision is made based on an **eval signal** computed from the trace itself: a judge score below threshold, a guardrail trip, a refusal on benign traffic, a novel jailbreak pattern. Signal-driven sampling can be implemented as a tail-sampling policy (if the signal is computable at Collector time) or as a re-ingest pass (if the signal comes from a mod-107 judge that runs post-hoc). It is the highest-value policy per dollar for eval work — the traces you keep are, by construction, the ones that will actually move an SLI.

### The three, working together

The pattern that most teams end up on:

1. **Head sampling at a low base rate** — 1–10% depending on volume — as a **representative population** for aggregate SLIs.
2. **Tail sampling that keeps 100%** of the tails: errors, guardrail trips, high-cost traces, high-latency traces, canary arms.
3. **Signal-driven re-sampling** for the judge loop: on the head-sampled population, run a cheap scorer; on scorer hits (low quality, over-refusal, groundedness fail), promote the trace to the full-fidelity store and route to mod-109 human review.

The base rate feeds the aggregate SLI numbers (share of judged summaries above threshold, cost per session). The tails feed the debug workflow (chapter 05). The signal loop feeds the online-eval loop (mod-107). All three coexist in one Collector config.

### The three scrubbing positions

There are three places PII can be removed from a trace:

**At the application.** Before you set the attribute on the span. Your instrumentation wrapper is the first and best line of defense. Rule: any payload that carries user identity or content passes through a scrubber before `set_attribute`.

**At the Collector.** The `attributes` processor in `opentelemetry-collector-contrib` (<https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor>) supports `hash`, `delete`, and pattern-matched actions on attribute values; the `redaction` processor (<https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor>) redacts by regex patterns. This runs in your VPC before the trace ships to a backend — the last line of defense before payload leaves your control.

**At the backend.** Post-hoc redaction after the trace has already been stored on the vendor side. Worst option: the raw payload has already been on the wire and at rest.

**Recommendation**: application + Collector. Defense in depth. The Collector run is the compliance-critical one — treat it like an egress firewall.

### Common scrubbers

- **Regex allowlists** — cheap, deterministic, easy to review. Good for structured PII (emails, phone numbers, credit-card-like numeric strings) and for scanning attribute keys.
- **Microsoft Presidio** — <https://microsoft.github.io/presidio/>. Named-entity PII detection for many entity types; usable as a library at the application layer or as a service the Collector calls out to. Configurable analyzer + anonymizer.
- **Cloud DLP APIs** — Google Cloud DLP, Amazon Macie, etc. Use if your surface already runs in that cloud; watch cost.

Pick one for text scrubbing, plus a small regex sweep for common structured strings you know appear in your prompts. Cite the scrubber in your policy so an audit can trace redaction outcomes back to a rule.

### Hashing user ids

From chapter 04: HMAC-SHA256 of the raw user id, keyed with a per-tenant salt held in a KMS. Sketch (Python):

```python
import hmac
import hashlib

def user_hash(raw_user_id: str, tenant_salt: bytes) -> str:
    mac = hmac.new(tenant_salt, raw_user_id.encode("utf-8"), hashlib.sha256)
    return "u_" + mac.hexdigest()[:16]

# The tenant salt is fetched from KMS at process start; never logged.
salt = kms.get_secret(f"tenant/{tenant_id}/user-hash-salt")
```

Never commit the salt. Rotate periodically; keep a rotation log so mod-109 can still resolve historical traces. Under GDPR (Regulation (EU) 2016/679, Art. 4(5)), the hash is **pseudonymous** — personal-data obligations still apply, and the residual privacy risk should be enumerated in the surface's governance plan.

### Retention

Traces are personal data if they contain the user's raw or pseudonymous id. A retention statement is part of the policy. Structure — the specific numbers depend on your compliance frame and business context:

- **Hot store** (fully searchable, backend UI-facing) — N days. Typical: 7–30 days.
- **Warm store** (compressed, queryable via ETL) — M days. Typical: 30–90 days.
- **Cold store** (blob archive, retrieval on demand) — up to a policy-defined retention window.
- **Right-to-erasure** procedure: given a user id, produce the list of traces across tiers and purge. The procedure must be tested annually (mod-112 governance hand-off).

Reference-only: NIST SP 800-122 (<https://csrc.nist.gov/publications/detail/sp/800-122/final>) for PII handling guidance, and GDPR Art. 4(5) for pseudonymisation. The Governance-family peer track owns the sign-off shape; this module produces the technical evidence.

### The residual-risk statement

Every sampling + PII policy leaves a residue you cannot measure:

- **Dropped tails.** Traces that were head-sampled out and were not on any tail policy are gone forever. If a novel failure class did not trip an existing tail rule, you learn about it late.
- **Scrubbed content.** A judge running on a scrubbed trace sees redacted email addresses; a substring-match groundedness check may fail because the retrieved doc had a name the reply span no longer has.
- **Hashed users.** You cannot reconstruct which specific user hit which specific trace without a KMS lookup, and cross-tenant analyses are by-construction impossible.

Name the residual explicitly. The recommendation for the "I need the raw payload" case: keep a **separately gated, small, consented, unredacted store** of traces used for repro (mod-109 review). Access is audited, retention is short, and consent is explicit.

## A Collector recipe

The following is a directional skeleton — not a production config. Validate with `otelcol-contrib validate` and cross-check attribute names against your schema (chapter 04).

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:
    timeout: 1s

  # PII scrubbing at the Collector — defense in depth.
  attributes:
    actions:
      # Drop attribute names that should never leave the VPC.
      - key: "app.raw_email"
        action: delete
      # Hash any lingering user-id keys.
      - key: "user.id"
        action: hash
      # Redact string content with a regex pattern (kept small; cite in policy).
      - key: "input.value"
        action: update
        pattern: "\\b[\\w.+-]+@[\\w-]+\\.[\\w.-]+\\b"
        value: "<email>"

  # Tail sampling.
  tail_sampling:
    decision_wait: 10s
    policies:
      # Always keep errors.
      - name: keep_errors
        type: status_code
        status_code: { status_codes: [ERROR] }

      # Always keep guardrail trips.
      - name: keep_guardrail_trips
        type: string_attribute
        string_attribute:
          key: app.guardrail_verdict
          values: [block, rewrite]

      # Always keep the canary arm during rollout.
      - name: keep_canary
        type: string_attribute
        string_attribute:
          key: experiment.arm
          values: [reply-model-v3]

      # Always keep slow tails (adjust threshold per surface SLO).
      - name: keep_slow
        type: latency
        latency: { threshold_ms: 8000 }

      # Rate-limited random slice as the representative population.
      - name: base_rate_random
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }

exporters:
  otlphttp/phoenix:
    endpoint: "https://<phoenix-host>/v1/traces"
    headers: { ... }

service:
  pipelines:
    traces:
      receivers: [otlp]
      # Scrub before sampling — never let raw PII into the tail-sampling decision cache.
      processors: [attributes, tail_sampling, batch]
      exporters: [otlphttp/phoenix]
```

Two ordering rules worth stating up front:

1. **Scrub before tail-sample.** The tail-sampling processor buffers raw traces in memory. If a redaction bug ships payload downstream, the tail buffer is one of the places you should not have looked.
2. **Set the head-sampling rate at the SDK** (`TraceIdRatioBased`), not at the Collector. Head-sampling at the Collector saves storage but does not save your app's serialisation or egress; the point of head sampling is to reduce work up front.

### Budget math

The policy has to fit inside a dollar cap. The math is elementary, but writing it down is what makes the trade-offs real. For a fictional surface at 1,000 requests / minute:

- **Base rate.** 5% head = 50 traces / minute = ~2.16M traces / month.
- **Tail additions.** Assume `errors + guardrail trips + canary + slow tail` add ~2% of total traffic on top → ~0.86M traces / month.
- **Total kept.** ~3.02M traces / month.
- **Storage.** Assume $X per M spans stored on your backend. If a trace averages 6 spans, that is ~18M spans / month → **~18X dollars**. Fill in `$X` from vendor docs.
- **Judge scoring.** Assume you score $Y per judged trace on `~10%` of the kept traces → 0.3M judged traces × Y → **~0.3M·Y dollars**.
- **Blob store.** Payload overflow (chapter 04) at ~50 KB average for surge traces → cost per GB per month × total GB. Cheap.

Do not commit to concrete `X` and `Y` in this doc — they change monthly. Do commit to *showing the arithmetic*, so your budget is auditable and your Governance-family peer can review it.

### SLI-preservation check

Under any policy, run this check per SLI from mod-101:

- **Faithfulness.** Judge runs on ~10% of kept traces. That is enough for a weekly SLO with confidence bands wider than a single-day comparison — say so.
- **Tool-argument correctness.** Errors are 100%-sampled by `keep_errors`. If the failure is not an OTel error status, the base rate catches ~5%. If either fires, the SLO is computable.
- **End-to-end p95 latency.** `keep_slow` on latency ≥ 8s guarantees the tail is captured; the median is computable from the base rate.
- **Guardrail refusal rate.** 100% of trips are kept by `keep_guardrail_trips`; the denominator (traffic without a trip) is estimable from the base rate.

If any SLI drops below acceptable confidence, either raise the base rate or add a targeted tail policy. Do the math, don't guess.

## Example: policy for the support-agent surface

Concrete policy for the surface used in chapters 04 and 05, assuming ~1,000 requests / minute and a compliance-sensitive tenant:

- **Sampling**
  - Head: 5% probabilistic at the SDK.
  - Tail: 100% of `status=Error`; 100% of `app.guardrail_verdict∈{block,rewrite}`; 100% of `experiment.arm=reply-model-v3` for the duration of the canary; 100% of `duration_ms >= 8000`; base-rate random slice at 5%.
  - Signal-driven: mod-107 judge on 10% of kept traces; any trace with `app.judge_score < 3` (on a 1–5 rubric) is promoted to the review store and routed to mod-109.

- **Scrubbing**
  - At the application: HMAC-SHA256 hash for `user.id`; email regex over `input.value`, `output.value`, `llm.input_messages.*.content`, `llm.output_messages.*.content`; drop `Authorization` and cookie headers on TOOL spans.
  - At the Collector: mirror of the application scrubs plus a Presidio pass on `input.value` and `output.value`; hash of any residual `user.*` key; deny-list on `app.raw_*` keys.

- **Hashing**
  - `user.id = "u_" + HMAC-SHA256(raw_id, tenant_salt).hex()[:16]`; salt in KMS.

- **Retention**
  - Hot 14 days, warm 60 days, cold 180 days; right-to-erasure procedure documented in the surface's `RUNBOOK.md`; tested quarterly.

- **Budget math** — worked in the surface's `BUDGET.md` alongside vendor unit prices refreshed monthly.

- **SLI preservation** — worked in `SLI_PRESERVATION.md`; each of the four SLIs (faithfulness, tool-argument correctness, latency p95, guardrail refusal rate) computable under the policy.

- **Residual risk** — traces dropped by head sampling and *not* kept by any tail policy are unrecoverable; a small consented unredacted store gated to mod-109 covers the repro requirement; cross-tenant analyses are structurally impossible.

- **Governance sign-off** — deferred to peer role `ai-evaluation-engineer` (level 25, Governance family).

## Summary

- Three sampling positions: **head** (SDK, cheap, blind to the tail), **tail** (Collector, buffered, sees the whole trace), **signal-driven** (eval-signal-driven, most valuable per dollar for eval work). Use all three together.
- Two scrubbing positions worth deploying: **application** (before the payload lands on a span) and **Collector** (before the payload leaves the VPC). Backend redaction is a fallback, not a plan.
- Recommended scrubbing stack: regex + Microsoft Presidio for text; HMAC-SHA256 with per-tenant KMS salt for user ids; deny-list for known-sensitive attribute keys.
- Order matters: **scrub before tail-sample**, so raw PII never enters the tail-sample buffer.
- Retention has hot / warm / cold tiers, plus a tested right-to-erasure procedure. Reference NIST SP 800-122 and GDPR Art. 4(5); defer the sign-off to the Governance-family peer.
- Budget math and SLI-preservation check are part of the policy artefact, not implicit — write them down so an auditor can read them.
- The residual risk (dropped tails, scrubbed content, unmeasurable classes) is stated openly; the mitigation is a small consented unredacted store, gated to the review team.

This chapter closes the module. The next module (mod-103) uses these traces to evaluate agent trajectories.
