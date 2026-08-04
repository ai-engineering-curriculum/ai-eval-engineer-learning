# exercise-05: Sampling and PII Policy

**Estimated effort:** 3 hours

## Objective

Produce the **sampling + PII scrubbing policy** for the surface you instrumented in exercises 01–04, deploy it in an OpenTelemetry Collector, and verify with a scripted drill that (a) the head-and-tail policy keeps what you claim it keeps, (b) the scrubbers strip what you claim they strip, and (c) each SLI from your `mod-101` criteria table is still computable under the policy. The deliverable is the artefact chapter 06 describes: a written policy + a working Collector config + a set of drill results + a residual-risk statement, in a shape a Governance-family peer (peer role `ai-evaluation-engineer`, level 25) could sign off.

## Prerequisites

- Chapter 06 of this module (and, ideally, chapter 04's schema doc).
- The schema doc from exercise-03 (or chapter 04's support-agent baseline).
- The Collector from exercise-02 running and fanning out to at least one backend.
- Microsoft Presidio installed locally (`pip install presidio-analyzer presidio-anonymizer`, plus a spaCy model — see <https://microsoft.github.io/presidio/installation/>), *or* an equivalent scrubber you can justify.
- Your `mod-101` criteria table — the SLIs are the input to the "SLI preservation" check.

## Requirements

Produce a directory `mod-102/exercise-05/` containing five files.

### 1. `POLICY.md`

The human-readable policy. Structure required; the numbers are yours.

- **Surface + traffic assumption.** One paragraph naming the surface, its `app.surface` value, an assumed traffic rate (requests / min at steady state, peak / mean), and the compliance frame you are working under (GDPR-EU / US-only / regulated industry — you choose, but state it).
- **Sampling.**
  - **Head:** the SDK-level sampler (`ParentBased(TraceIdRatioBased(p))`) with `p` justified. State the fraction and why.
  - **Tail:** every policy name in the Collector, one line each, with the attribute / condition. Cover at minimum: errors, guardrail trips, canary arm, slow tail (with threshold matched to your `mod-101` latency SLO), and a probabilistic base rate.
  - **Signal-driven:** how the `mod-107` judge results feed back to promote a trace to the review store. State the score threshold and the routing target (`mod-109` queue name).
- **Scrubbing.**
  - **At the application:** the exact list of attribute keys the app wrapper redacts before `set_attribute`, the redaction pattern (`<email>`, `<phone>`, etc.), and the reason each key is on the list.
  - **At the Collector:** mirror of the application scrubs plus at least one *additional* stage (Presidio, custom regex, cloud DLP) that catches text the application scrubber missed. Cite the processor names.
  - **User-id hashing:** the HMAC-SHA256 recipe with per-tenant salt in KMS.
- **Retention.** Hot / warm / cold days per tier + right-to-erasure procedure (steps + expected recovery time).
- **SLI-preservation check.** A table with one row per SLI from your `mod-101` table. Columns: `SLI`, `sampled-population source` (which sampling policy captures it), `expected weekly sample size`, `confidence-band note` (one sentence), `verdict` (`OK` / `needs-a-targeted-tail-policy`).
- **Budget math.** Concrete arithmetic in the shape of chapter 06: traces / month kept, spans / month, judge scores / month, blob-storage GB / month. Vendor unit prices (`$X`, `$Y`) may be TBD but the arithmetic must be present.
- **Residual risk.** Three named items with mitigations: dropped tails not covered by any policy; scrubbed content that limits an offline judge; hashed users that block cross-tenant analysis. Explicitly state the mitigation (small consented store, judge-side handling, structural impossibility).
- **Governance sign-off owner.** Name the peer role and how you will hand off (link, artefact, expected turnaround).

### 2. `collector.yaml`

A working Collector config that implements the policy. Requirements:

- Extends exercise-02's `collector.yaml` — do not fork.
- Uses the `attributes` and/or `redaction` processors from `opentelemetry-collector-contrib` for the Collector-side scrubbing, with concrete `key`, `pattern`, and `action` fields matching Section "Scrubbing → At the Collector" above.
- Uses the `tail_sampling` processor with one `policies` entry per rule in Section "Sampling → Tail" above.
- Orders the processor pipeline `attributes → tail_sampling → batch`. State in a comment above the pipeline block *why* that order (chapter 06: scrub before tail-sample so raw PII never enters the tail buffer).
- Validates cleanly with `otelcol-contrib validate --config=collector.yaml`.

### 3. `drill.py` + `drill-results.md`

A script + a report from running it against your instrumented service through the Collector.

- **`drill.py`** — sends ~200 synthetic requests: ~150 normal, ~30 that should trip specific tail policies (errors, guardrail trips, canary arm, slow requests), and ~20 with seeded PII in the input (`someone@example.com`, phone number, credit-card-like digit strings, a raw user id).
- **`drill-results.md`** — a table with one row per tail-sampling policy in your `POLICY.md`, columns: `policy`, `expected keeps` (from the seeded traffic), `observed keeps in backend`, `verdict`. Include a second table with one row per PII pattern: `pattern`, `count in raw payload`, `count in backend after scrub`, `verdict`. Verdict is `OK` only when observed matches expected exactly; near-misses are `investigate` with a one-line hypothesis.

### 4. `sli-preservation-check.md`

A per-SLI analysis showing that under this policy, each SLI is still computable. This is the same table as in `POLICY.md` §"SLI-preservation check", *filled with real numbers from the drill's kept traces* rather than expectations. If any SLI's kept-sample size falls below what a weekly SLO comparison needs, either raise the base rate or add a targeted tail policy — show the change.

### 5. `residual-risk-statement.md`

A short (≤ 400 words) document explicitly for the Governance-family peer:

- **Three residual risks**, each with: what the policy cannot see, why the mitigation is acceptable (or why it is not), and what would trigger a revisit.
- **What is escalated.** A bulleted list of decisions this policy defers to Governance sign-off (retention windows, cross-region export, consented-store scope). Include the peer role `ai-evaluation-engineer` (level 25) as the sign-off owner and the target sign-off date.

## Starter guidance

- Write `POLICY.md` first in prose, before you touch YAML. If a paragraph is hard to write, the config for it is going to be worse.
- The exact tail-sampling policy names in `opentelemetry-collector-contrib` change occasionally — cross-check names against the current README (`resources.md` links it). Do not copy chapter 06's snippet verbatim.
- Presidio's default analyzers cover common entity types, but PII detection is imperfect. Do not claim in `POLICY.md` that the policy *guarantees* no PII escapes — say "reduces to residual per drill results" and cite the drill.
- Run `drill.py` against a Collector that is currently forwarding to your backend, and *read the backend* to check what actually landed. Do not trust that the config does what you wrote in `POLICY.md`; verify.
- The SLI-preservation check often surfaces the need to raise a base rate or add a targeted tail policy. If yours does not, treat that as a warning — either your SLIs are unusually forgiving or the drill is not exercising them. Say so.
- Do not commit real API keys, real user emails, or real production traces. The whole drill is synthetic.

## Acceptance criteria

You are done when:

- `POLICY.md` covers every required section with concrete values (not "TBD" except on vendor unit prices).
- `collector.yaml` validates and, when run, produces the sampling and scrubbing behaviour promised in `POLICY.md` on the drill traffic.
- Every tail-sampling policy row in `drill-results.md` has an `OK` verdict *or* an `investigate` note explaining the mismatch.
- Every seeded PII pattern in `drill-results.md` shows zero (or a documented, tiny near-miss) in the backend.
- `sli-preservation-check.md` uses observed drill numbers, not expectations, and every SLI reaches the confidence bar you claim in `POLICY.md`.
- `residual-risk-statement.md` names three specific residuals with mitigations and identifies the Governance sign-off owner.

## Stretch goals

- Add a **PII red-team** step: hand-write ten inputs that carry unusual PII (a foreign-format phone number, a national id, a physical address split across two fields) and see how many the Presidio + regex stack catches. Report the miss rate in `drill-results.md` under a separate table.
- Add a **rotation drill**: rotate the user-id salt, re-run 20 requests, and confirm that (a) new hashes differ from old, (b) `mod-109` can still resolve the previous window via the salt rotation log.
- Add a **cost projection** at 10× and 100× traffic. Do the arithmetic in `POLICY.md` §"Budget math" and note the crossover point where a policy adjustment (raise tail threshold, drop base rate, move to warm store faster) makes sense.
- Add a **`right_to_erasure.py`** stub: given a `user.id`, produce the list of trace ids across the three retention tiers and dry-run the purge. Cite what your backend's actual delete API looks like from its docs.

## What this exercise does *not* cover

You are not writing the online-eval judge itself (that is `mod-107`), and you are not standing up the eval-data platform's cold store (that is `mod-110`). You are also not producing the Governance sign-off — you are producing the artefact the Governance-family peer signs off *on*. The peer's own workflow is theirs; your job is to hand them a policy they can review in one sitting.
