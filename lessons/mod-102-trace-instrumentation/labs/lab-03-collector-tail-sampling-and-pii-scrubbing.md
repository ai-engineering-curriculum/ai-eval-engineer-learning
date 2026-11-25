# lab-03: OTel Collector with Tail Sampling and PII Scrubbing

**Estimated duration:** 75–90 minutes
**Pairs with:** chapter 06 of this module, and chapter 03 for the Collector transport shape.

## Objective

Chapter 06 sets the rules: scrub before you tail-sample; keep the tails that matter (errors, guardrail trips, canary arms, slow traces) at 100% and let the rest flow through a probabilistic base rate; hash user ids with a per-tenant salt; verify behaviour with a drill. This lab stands up a working **OpenTelemetry Collector** locally with those processors in order, sends a 25-span drill through it, and checks a sink file to confirm the Collector (a) kept what the policy says it keeps and (b) scrubbed what the policy says it scrubs.

The goal is to make the chapter-06 policy concrete in a Collector config you can `otelcol-contrib validate` and point at a drill. Everything runs on your laptop — no vendor backend, no production data. The `file` exporter writes kept traces to a local JSONL file you grep against.

This lab does *not* duplicate exercise-05: that exercise writes the whole `POLICY.md`, the `collector.yaml` extending exercise-02's, the `drill.py`/`drill-results.md`, the `sli-preservation-check.md`, and the `residual-risk-statement.md` for the surface you own. This lab is the warm-up — one Collector, one drill, one results file — so that when exercise-05 asks you to do it end-to-end you already know the Collector works.

## Prerequisites

- Chapter 06 read (sampling positions, two-layer scrubbing, Collector recipe, ordering rule). Chapter 03 helpful because the Collector is the portable transport it describes.
- Python 3.10+ (`python3 --version`).
- **Docker** (or Podman) so you can run the `otel/opentelemetry-collector-contrib` image — the `tail_sampling`, `attributes`, and `redaction` processors ship in the `-contrib` distribution, not the core one. If you prefer the binary, download `otelcol-contrib` from <https://github.com/open-telemetry/opentelemetry-collector-releases/releases> and use it in place of the `docker run` lines below; the config is identical.
- Network: localhost only. The Collector binds OTLP-HTTP on `:4318`; the drill script posts to it.
- Lab-01 is helpful context (same instrumentation shape) but not a strict prerequisite.

## Layout you will build

```
mod-102-lab-03/
  collector.yaml
  drill.py
  trace_sink.jsonl      # written by the Collector's file exporter
  drill-results.md
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-102-lab-03 && cd mod-102-lab-03
touch trace_sink.jsonl        # create empty file so Docker can bind-mount it
```

### Step 2 — Write the Collector config (`collector.yaml`)

The processor order is the one chapter 06 fixes: `attributes → redaction → tail_sampling → batch`. The attributes processor hashes `user.id` and deletes known-sensitive keys; the redaction processor runs a regex over attribute *values* to strip emails (defense-in-depth for text the application layer missed); the tail_sampling processor keeps errors, the canary arm, slow traces, and a 10% base rate; the file exporter writes kept traces as JSONL into the sink.

Create `collector.yaml`:

```yaml
# Chapter 06's recipe, applied to a single local sink.
#
# Order matters: scrub before tail-sample so raw PII never enters the
# tail-sampling buffer (chapter 06 §A Collector recipe).
#
# The exact processor field names are checked against the current
# opentelemetry-collector-contrib README — syntax drifts, validate with
# `otelcol-contrib validate --config=collector.yaml` before deploying.

receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  # Application-layer scrubbing mirror: hash user.id, drop known
  # raw-PII keys. Chapter 06 §The three scrubbing positions names this
  # as the Collector-side defense-in-depth layer.
  attributes:
    actions:
      - key: user.id
        action: hash
      - key: app.raw_email
        action: delete

  # Text-value scrubbing. Blocks regex patterns inside attribute values;
  # kept small and reviewable per chapter 06 §Common scrubbers.
  redaction:
    allow_all_keys: true
    blocked_values:
      - '[\w.+-]+@[\w-]+\.[\w.-]+'

  # Tail sampling. One policy per chapter-06 keep-rule.
  tail_sampling:
    decision_wait: 5s
    num_traces: 100
    expected_new_traces_per_sec: 10
    policies:
      - name: keep-errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: keep-canary
        type: string_attribute
        string_attribute:
          key: experiment.arm
          values: [canary]
      - name: keep-slow
        type: latency
        latency:
          threshold_ms: 2000
      - name: base-rate
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

  batch:
    timeout: 1s

exporters:
  # File sink writes kept traces as JSONL — one line per exported
  # batch. The drill script reads this to assert the policy did what it
  # was supposed to do.
  file:
    path: /etc/output/trace_sink.jsonl

service:
  pipelines:
    traces:
      receivers: [otlp]
      # Scrub before tail-sample — chapter 06's ordering rule.
      processors: [attributes, redaction, tail_sampling, batch]
      exporters: [file]
```

Validate it:

```bash
docker run --rm -v "$PWD/collector.yaml:/etc/otelcol-contrib/config.yaml" \
  otel/opentelemetry-collector-contrib:latest validate \
  --config=/etc/otelcol-contrib/config.yaml
```

**Expected:** the command exits `0`. If it exits non-zero with a processor-field error, the `opentelemetry-collector-contrib` version's field names have drifted — cross-check against the current README (`processor/tailsamplingprocessor`, `processor/attributesprocessor`, `processor/redactionprocessor`) and patch the config. Chapter 06 flags this as a syntax surface that moves.

### Step 3 — Run the Collector

Start it in the background with the sink file bind-mounted so the host can grep it:

```bash
docker run -d --name mod102-lab03 \
  -p 4318:4318 -p 4317:4317 \
  -v "$PWD/collector.yaml:/etc/otelcol-contrib/config.yaml" \
  -v "$PWD/trace_sink.jsonl:/etc/output/trace_sink.jsonl" \
  otel/opentelemetry-collector-contrib:latest \
  --config=/etc/otelcol-contrib/config.yaml

# Confirm it's up — expect "Everything is ready. Begin running and processing data."
docker logs mod102-lab03 2>&1 | tail -n 20
```

**Expected:** the Collector logs that the OTLP receivers started on 4317 and 4318 and that the pipeline is running. If the container died, `docker logs mod102-lab03` will show the parse / validation error on the config.

### Step 4 — Install the OTel SDK and HTTP exporter

The drill script sends spans via OTLP-HTTP, so you need the HTTP exporter package in addition to the SDK:

```bash
pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
```

### Step 5 — Write the drill script (`drill.py`)

The drill sends 25 spans in four cohorts chosen so each tail-sampling policy and each scrubber has clearly countable traffic:

- **10 baseline** spans — no special attributes, no PII. Expect the `base-rate` 10% policy to keep ~1.
- **5 error** spans — `status=ERROR`, clean content. Expect `keep-errors` to keep all 5.
- **3 canary** spans — `experiment.arm=canary`, clean content, no error, no slow. Expect `keep-canary` to keep all 3.
- **2 slow** spans — latency > 2s, no error, no canary. Expect `keep-slow` to keep both.
- **5 PII-seeded** spans — email in `input.value`, raw user id in `user.id`, a `app.raw_email` attribute, marked as ERROR so they are definitely kept by `keep-errors`. Expect the sink file to contain zero emails and zero raw user ids after Collector scrubbing.

Create `drill.py`:

```python
# drill.py
"""Drill traffic for chapter 06's Collector recipe.

Sends 25 spans in five cohorts, each sized so the tail-sampling policy
and the scrubbers have clearly countable expected outcomes."""

import time

from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.trace import SpanKind, Status, StatusCode

COLLECTOR = "http://localhost:4318/v1/traces"
TENANT = "lab03"

# One global tracer provider pointed at the Collector.
provider = TracerProvider(
    resource=Resource.create({"service.name": "lab03-drill"})
)
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint=COLLECTOR))
)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("mod-102.lab-03.drill")


def _emit(
    name: str,
    *,
    error: bool = False,
    canary: bool = False,
    slow_ms: int = 0,
    pii: bool = False,
    tag: str = "",
) -> None:
    with tracer.start_as_current_span(name, kind=SpanKind.INTERNAL) as span:
        attrs = {
            "openinference.span.kind": "AGENT",
            "app.surface": "lab03-stub",
            "session.id": f"sess_{name}",
            "experiment.arm": "canary" if canary else "control",
            "app.cohort_tag": tag,
        }
        if pii:
            attrs["user.id"] = "alice@example.com"           # raw id — must be hashed by Collector
            attrs["app.raw_email"] = "alice@example.com"     # must be deleted by Collector
            attrs["input.value"] = "contact me at bob@example.com about my ticket"
        else:
            attrs["user.id"] = f"u_{name}"                    # already a stable opaque id
            attrs["input.value"] = "ordinary question with no email in it"
        span.set_attributes(attrs)
        if slow_ms > 0:
            time.sleep(slow_ms / 1000.0)
        if error:
            span.set_status(Status(StatusCode.ERROR, "drill-seeded error"))


def main() -> None:
    # Baseline — probabilistic policy keeps ~1 of these.
    for i in range(10):
        _emit(f"baseline_{i}", tag="baseline")

    # Errors — keep-errors keeps all 5.
    for i in range(5):
        _emit(f"error_{i}", error=True, tag="error")

    # Canary — keep-canary keeps all 3.
    for i in range(3):
        _emit(f"canary_{i}", canary=True, tag="canary")

    # Slow — keep-slow keeps both.
    for i in range(2):
        _emit(f"slow_{i}", slow_ms=2500, tag="slow")

    # PII — all marked ERROR so they survive to the sink, where you
    # grep for the scrubbing outcome.
    for i in range(5):
        _emit(f"pii_{i}", error=True, pii=True, tag="pii")

    # Flush so the Collector sees the spans before the script exits.
    provider.shutdown()


if __name__ == "__main__":
    main()
```

Run it:

```bash
python3 drill.py
# Give the Collector's tail_sampling decision_wait (5s) a moment to finish.
sleep 8
```

**Expected:** the script completes cleanly. Nothing prints — the drill is silent by design.

### Step 6 — Verify tail-sampling kept what the policy promised

Read the sink file and count spans per cohort by their `app.cohort_tag` attribute. The cohort tag is set to the same string as the keep rule, which makes the grep direct.

```bash
# How many total spans survived to the sink?
wc -l trace_sink.jsonl

# One count per cohort. The file exporter writes OTLP-JSON batches; the
# cohort tag appears verbatim because the attributes processor does not
# touch it.
for c in baseline error canary slow pii; do
  cnt=$(grep -o "\"app.cohort_tag\":\"$c\"" trace_sink.jsonl 2>/dev/null \
        || grep -o "\"stringValue\":\"$c\"" trace_sink.jsonl \
        | wc -l)
  echo "$c: $cnt"
done
```

**Expected (approximate):**

| Cohort | Sent | Expected kept | Why |
|---|---|---|---|
| baseline | 10 | ~1 (±1) | `base-rate` is probabilistic at 10% |
| error | 5 | 5 | `keep-errors` catches all |
| canary | 3 | 3 | `keep-canary` catches all |
| slow | 2 | 2 | `keep-slow` catches all |
| pii | 5 | 5 | PII spans were also marked ERROR so `keep-errors` catches them |

If the error / canary / slow / pii counts are below the "sent" column, one of the tail policies did not match — read the Collector logs (`docker logs mod102-lab03`) and check the attribute name against the policy's `key:` field. The probabilistic cohort is the only one allowed to vary run to run.

### Step 7 — Verify PII scrubbing stripped what the policy promised

Chapter 06's two-layer scrubbing claim is only honest if you can grep the sink file for the seeded strings and get zero hits.

```bash
echo "raw emails remaining in sink:"
grep -c '@example.com' trace_sink.jsonl || echo 0

echo "raw user id 'alice@example.com' remaining:"
grep -c 'alice@example.com' trace_sink.jsonl || echo 0

echo "raw-PII key 'app.raw_email' remaining:"
grep -c 'app.raw_email' trace_sink.jsonl || echo 0

echo "sample of user.id values after hashing:"
grep -o '"user.id"[^,}]*' trace_sink.jsonl | sort -u | head
```

**Expected:**

- `@example.com` count → **0** (redaction processor's regex caught both `alice@example.com` and the `bob@example.com` string in `input.value`).
- `alice@example.com` count → **0**.
- `app.raw_email` count → **0** (attributes processor `delete` action dropped the key).
- `user.id` values → either the SHA-256 hex digests the attributes processor produced from the raw strings, or (for non-PII spans) the opaque `u_<name>` values the drill set directly.

If any count is non-zero, the scrubber did not catch that pattern. Note the gap in `drill-results.md` as an `investigate` row — chapter 06's residual-risk statement is specifically for cases where a scrubber misses.

### Step 8 — Write `drill-results.md`

The report is short — one table per rule class plus a two-sentence note on what you would raise with a Governance-family peer if this were a real surface.

Create `drill-results.md`:

```markdown
# Drill results — lab-03

## Tail-sampling verdict

| Policy        | Expected kept | Observed kept | Verdict       |
|---------------|---------------|---------------|---------------|
| base-rate     | ~1 of 10      | <fill in>     | OK / investigate |
| keep-errors   | 5 of 5        | <fill in>     | OK / investigate |
| keep-canary   | 3 of 3        | <fill in>     | OK / investigate |
| keep-slow     | 2 of 2        | <fill in>     | OK / investigate |
| keep-errors (PII cohort) | 5 of 5 | <fill in>     | OK / investigate |

## PII scrubbing verdict

| Pattern              | Count in sent | Count in sink | Verdict |
|----------------------|---------------|---------------|---------|
| `@example.com` text  | 10            | <fill in>     | OK / investigate |
| `alice@example.com`  | 5             | <fill in>     | OK / investigate |
| key `app.raw_email`  | 5             | <fill in>     | OK / investigate |
| raw user id in user.id | 5           | <fill in>     | OK (hashed) / investigate |

## Residual risk (one sentence each)

- **Dropped tails.** 9 of 10 baseline spans were not kept — a novel failure class that only appeared in the baseline and did not trip errors, canary, or slow would be lost. Mitigation: raise the base rate, or add a signal-driven promotion from a judge (`mod-107`).
- **Scrubbed content.** Emails were masked to a redaction token; a judge that depends on the email string to reason about the complaint will see redacted text.
- **Hashed users.** Cross-tenant analysis is structurally impossible; historic traces require the salt rotation log to resolve.

Governance sign-off: deferred to peer role `ai-evaluation-engineer` (level 25).
```

Fill in the counts from steps 6 and 7. The `investigate` rows should include a one-line hypothesis — "the regex missed an aliased Unicode variant", "the sampling decision window closed too early", etc.

### Step 9 — Shut the Collector down

```bash
docker stop mod102-lab03 && docker rm mod102-lab03
```

### Step 10 — Checkpoint

You are done with this lab when every statement below is true.

- [ ] `otelcol-contrib validate --config=collector.yaml` exits `0` (check in step 2).
- [ ] The Collector container is running and the pipeline log line appears in `docker logs mod102-lab03`.
- [ ] `drill.py` posts 25 spans to `http://localhost:4318/v1/traces` without errors.
- [ ] `trace_sink.jsonl` contains between ~14 and ~17 kept span batches (5 error + 3 canary + 2 slow + 5 pii-but-error + 0–2 baseline).
- [ ] `grep -c '@example.com' trace_sink.jsonl` returns `0`.
- [ ] `grep -c 'app.raw_email' trace_sink.jsonl` returns `0`.
- [ ] `drill-results.md` has every row filled in with a verdict (`OK` or `investigate` + a one-line hypothesis).
- [ ] You can state the chapter-06 reason for `processors: [attributes, redaction, tail_sampling, batch]` ordering in one sentence: scrub before tail-sample so raw PII never enters the tail-sampling buffer.

## Where this hands off

- `collector.yaml` is the baseline exercise-05 asks you to extend with `mod-107` signal-driven promotion, retention tiers, a budget-math block, and an SLI-preservation check.
- The regex-only scrubbing here is the floor, not the ceiling. Chapter 06 recommends Presidio + regex for text; exercise-05's stretch goals include a PII red-team to measure the miss rate.
- The `file` exporter sink is the laptop stand-in for a real backend. Swap the exporter block for the OTLP exporter against Phoenix / Langfuse / Weave / Braintrust / LangSmith (chapter 03) and the rest of the config stays unchanged — that is the whole point of instrumenting against OTel once.
- The verdict table in `drill-results.md` is the shape a Governance-family peer wants before signing off: concrete numbers per policy, a named residual-risk statement, and a sign-off owner.
