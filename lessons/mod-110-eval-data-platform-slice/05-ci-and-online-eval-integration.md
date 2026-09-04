# CI and Online-Eval Integration: Queues, Retries, Idempotency, Quotas, Cost per Team

## Motivation

The plane exists to serve two triggers:

- **CI** — the mod-106 PR gate, deploy gate, and any per-canary stage gate. A PR-gate run is *bursty*: five commits land in ten minutes, each fires a run against the golden set. A deploy-gate run is *scheduled*: predictable, one per deploy. Both must return in a bounded time or the developer / release manager blocks.
- **Online eval** — the mod-107 sampled-trace loop. Runs are *sustained*: hundreds to millions per day at a fairly steady rate. Individual runs are small (usually one case) but the throughput is what matters.

The two triggers have opposite failure modes. A CI trigger cares deeply about *latency* (the developer is waiting) and mildly about *cost* (a bounded number of runs per PR). An online-eval trigger cares deeply about *cost* (the daily total is what shows up on the invoice) and mildly about *per-request latency* (an individual score can be a minute late without impact). If the plane treats them the same, one of them starves the other — usually the online loop starves under CI bursts, the drift monitor's window goes empty, and no one notices for a day.

The chapter's job is to make both triggers first-class, at their own priority, with rate-limits and retries that behave under vendor 429s, idempotency that survives duplicate triggers, quotas that let a runaway team be throttled without paging the eval team, and cost accounting that reads cleanly at the end of every month.

## Core concepts

### The two integration surfaces

**CI trigger.** The mod-106 PR / deploy gate calls the plane's control API:

```
POST /v1/runs
  body: EvalRunSpec
  headers:
    X-Cost-Centre: team-support/project-bot
    X-Trigger:     pr_gate
    X-Idempotency: pr:1234:commit:abc123:eval:support_bot_golden:sha256:aaaa...
    X-Priority:    class_a
```

The plane returns a `run_id` immediately. The gate polls `GET /v1/runs/{run_id}` (or subscribes to a webhook) until `status ∈ {succeeded, failed, throttled, cancelled}`, then reads the `run_manifest.json` and emits pass / fail per the mod-106 gate rule.

**Online-loop trigger.** The mod-107 sampler emits one `EvalRunSpec` per sampled trace (or per batch, when the runner supports batching). The trigger uses the same API but with a different priority:

```
X-Priority: class_c
X-Trigger:  online_loop
X-Idempotency: online:2026-09-04T14:03:00Z:trace:5b7e...:rubric:aaaa...
```

There is no polling loop for online-loop runs — the sampler fires and forgets; results land in the store via the runner adapter; the mod-107 aggregator picks them up.

The plane routes both to the same runner adapters, the same queue, the same cost accounting, and the same lineage store. The *only* thing the trigger difference produces is the priority class and the cost-centre routing.

### Priority classes

Three classes cover the common cases. Each class has a queue with a weighted-fair scheduler, a rate-limit envelope against the vendor, and a per-class SLO.

| Class | Trigger | Priority | Rate-limit share | Latency target | Cost target |
|---|---|---|---|---|---|
| A | `pr_gate`, `deploy_gate` | high | 50 % of vendor budget | p95 ≤ 5 min for a 500-case run | pay-per-use |
| B | Ad hoc, scheduled nightlies, back-fill | medium | 30 % of vendor budget | p95 ≤ 30 min | pay-per-use |
| C | `online_loop`, `shadow`, `canary_online` | low but sustained | 20 % of vendor budget | p95 ≤ 5 min per row | strictly capped monthly |

The classes are not first-come-first-served; they are *weighted-fair*. A class-A burst of 50 PR-gate runs cannot starve the class-C stream — the online loop keeps its 20 % share and the vendor rate-limit is never exhausted by CI. Symmetrically, a class-C storm (someone bumped the mod-107 sample rate 10× by mistake) cannot delay a class-A run past its 5-minute target.

The scheduler is small — a token bucket per class, with the bucket size scaled to the class's share of the vendor's per-minute rate limit. The exercise ships one; existing message-bus libraries (Redis, RabbitMQ, SQS with weighted consumers) also work if the org already runs one.

### Rate-limit envelope

Every vendor has a per-key rate limit. The plane holds a single vendor key per model per region and imposes a *plane-wide* envelope well below the vendor's stated cap — typically 70 – 80 % of stated cap — so that:

- Bursts of retries do not saturate the vendor and cause a global degradation.
- A cross-team run does not starve another team's run because the vendor throttled.
- The plane's own alerting on rate-limit health has room to fire before the vendor's does.

The envelope is enforced *in the plane*, before the request goes to the runner adapter. The runner adapter never sees a 429 from the vendor under normal load; a 429 that does happen is a signal that the plane's estimate of the vendor's cap is off. Chapter 06's platform-SLI dashboard alerts on this.

### Retries and backoff

Retries happen on two shapes of failure:

- **Retriable vendor failures.** 429 (rate-limit), 502 / 503 / 504 (transient upstream), TCP resets, judge-model 5xxs. Exponential backoff with full jitter, capped at three retries and 60 s max delay. The rate-limit envelope should prevent most 429s; those that get through are back-pressure signals.
- **Retriable runner failures.** The runner subprocess crashed, the runner's own retry pool exhausted, an unexpected shape parsing error. Same policy, but log the runner's full failure so the adapter maintainer can fix the underlying bug.

Non-retriable failures fail the run:

- Judge model returned a well-formed but useless response ("I can't evaluate this"). Log, flag for judge-tier review.
- The case body violates the schema (chapter 03's schema check should have caught it upstream, but a schema migration drift is possible). Log, flag the case for retirement or fix.
- The target model responded with a shape the chain does not expect. Log, flag as an app bug; escalate to the owning team.

The plane's job is to *classify* the failure and *account for the retries*. Every retry counts against the cost centre's cost — silently retrying is the fastest way to blow a monthly budget.

### Idempotency

The `X-Idempotency` header (also present as `EvalRunSpec.idempotency_key`) is what lets the plane deduplicate:

- **Same key, run already exists, still in progress.** Return the existing `run_id`; the trigger polls that.
- **Same key, run already exists, terminal.** Return the existing `run_id` and the existing `run_manifest.json`. The trigger sees the cached result immediately and does not re-pay.
- **Same key, run already exists, older than a configured window.** Configurable per class — often 24 h for class A (a PR retry a day later is legitimately a fresh run), infinite for class C (the same online-loop trace does not need re-scoring).
- **Different key, otherwise identical spec.** Two distinct runs — the plane trusts the trigger to decide.

Idempotency is what makes CI safe to retry, GitHub Actions re-runs safe, network flakes safe. Without it, a re-run doubles the cost and the eval-team scrambles.

The derivation rule for CI keys:

```
pr_gate:      pr:<pr_id>:commit:<sha>:eval:<eval_set_id>:rubric:<rubric_hash>
deploy_gate:  deploy:<deploy_id>:eval:<eval_set_id>:rubric:<rubric_hash>
online_loop:  online:<sample_bucket>:trace:<trace_id>:rubric:<rubric_hash>
scheduled:    schedule:<cron_id>:window:<yyyy_mm_dd>:eval:<eval_set_id>:rubric:<rubric_hash>
manual:       user:<uid>:spec_hash:<spec_hash>:ts:<yyyy_mm_dd_hh_mm>
```

The rule is: **the same trigger context must produce the same key.** A random per-request UUID defeats idempotency.

### Cost accounting per team

Every run has a `cost_centre` (usually `team_id/project_id`). Every judge call, every model call, every vendor invoice line, every retry, every cached vs live scoring is attributable to it.

The cost pipeline:

- **Per-row accounting.** Every `EvalResultRow.judgement.cost_usd` is derived at runner-adapter time from the vendor's token count and the plane's `pricing_snapshot` for the `judge_model_snapshot`. A `pricing_snapshot_id` column ties the row to the price sheet in force when the run happened — this lets you re-cost historic runs when a price sheet changes (or a mis-priced snapshot is corrected).
- **Per-run roll-up.** The `eval_runs.cost_usd` is the sum of the run's rows plus the adapter's fixed overhead (Braintrust's project storage cost per run, Weave's workspace cost per experiment, etc.). Recorded when the run terminates.
- **Per-team roll-up.** A materialised view (or a nightly job) aggregates `eval_runs.cost_usd` grouped by `cost_centre` and `week` / `month`. This is the number the invoice reconciles against.
- **Invoice reconciliation.** Monthly, the plane's per-team roll-up is joined against the vendor's invoice line items. Discrepancies > 5 % are a signal — usually a `pricing_snapshot` drift, sometimes a missed run. The reconciliation is a script, not an eyeball comparison.

The `pricing_snapshot` deserves a specific mechanism. Vendor prices change (usually favourably, sometimes not). A price change is a schema-preserving update:

- The plane holds `pricing_snapshots(snapshot_id, effective_from, effective_to, model_snapshot, unit_cost_input, unit_cost_output, currency)`.
- On a price change, the plane inserts a new snapshot with `effective_from = now()` and updates the previous snapshot's `effective_to`. New runs use the new snapshot; historic runs keep their snapshot pointer.
- The reconciliation script uses the effective snapshot per run's `started_at`.

### Quotas and enforcement

A quota is a monthly cap on `cost_centre.cost_usd`. Two enforcement points:

- **Soft at 80 %.** The plane emits a warning to the cost centre's Slack channel (or the org's cost-alerting surface). The next spec that would be dispatched includes a header `X-Quota-Warning: 82%`; the CI job that submitted it can render the warning to the developer.
- **Hard at 100 %.** The plane refuses new runs from that cost centre with an HTTP 402 and `X-Quota-Reason: cost centre X exceeded monthly quota; contact <owner> to raise or wait for reset`. In-flight runs finish; the online-loop sampler for that cost centre pauses.

Two escape hatches for the hard cap:

- **Override token.** A cost-centre owner can mint a *time-limited* override token (`X-Quota-Override: tok_...`) that raises their quota temporarily. The token has an expiry (24 h typical), an added-cap-amount, an issuer, and a reason. Every use is audited; the audit is visible to finance.
- **Emergency lift.** The eval team's on-call can lift a cap in an incident (a canary regression must be re-scored; the cap on the safety team's cost centre is blocking). The lift is time-boxed and requires an incident id.

The pattern the exercise implements is deliberately *fail closed*. The alternative — "the quota is advisory; runs never actually stop" — has been observed to produce vendor invoices multiple times the intended cap, at which point the finance conversation becomes an escalation. Chapter 06's error-budget mechanic is what makes fail-closed sustainable.

### The two artefacts the plane emits back to the trigger

CI reads two artefacts to decide pass / fail and to render a PR comment.

**`gate_result.json`.** The pass / fail per the mod-106 gate rule, computed *inside the plane* from the `eval_results` rows and the mod-106 threshold config the plane fetched from the same repo. Shape:

```json
{
  "run_id": "run_01JAV8...",
  "gate_pass": true,
  "gate_config_hash": "sha256:dddd...",
  "metrics": [
    {"metric": "faithfulness", "score": 0.87, "threshold": 0.80, "cohort": "aggregate", "status": "pass"},
    {"metric": "safety_refusal", "verdict_rate": 0.0, "threshold_max": 0.001, "cohort": "aggregate", "status": "pass"}
  ],
  "cost_centre": "team-support/project-bot",
  "run_cost_usd": 4.32,
  "cost_centre_month_to_date_usd": 812.47,
  "cost_centre_month_quota_usd": 2000.00
}
```

**`run_manifest.json`.** The chapter 04 artefact: run status, runner, timings, cost, artefact URIs. The gate reads this to link back to Braintrust / Weave / Langfuse from the PR comment.

The plane emits both to a stable per-run URL (`GET /v1/runs/{run_id}/artefacts/*.json`) and to an object-store path the CI can download. Both paths are stable for the run's retention window; changing them is a schema migration.

### Where the mod-107 online loop reads and writes

The mod-107 sampler fires a `EvalRunSpec` for every sampled trace. The plane treats each as a class-C run. Two rules:

- **The plane returns the `run_id` synchronously; results land asynchronously.** The sampler does not block on the score. This is what keeps the sampler's own throughput independent of the runner adapter's speed.
- **The scored-row schema from mod-107 chapter 02 is a projection.** The `eval_results` row is the source of truth; the mod-107 scored-row query is a view over it filtered by `source='online_loop'`. The mod-107 aggregator reads the view; no separate table.

This is the property that makes the mod-107 scored-row store and the mod-108 OWASP report and the mod-111 cost-quality dashboard *the same query* against the same underlying table.

### Failure modes and their runbooks

Every trigger integration ships with two runbooks. Chapter 06 fully specifies the SLIs; here are the alert shapes.

- **`ci_gate_timeout`.** The CI job called `POST /v1/runs` and never got a terminal `status` before its deadline. Owner: eval team. First step: check the plane's queue depth (class-A queue backed up? or the runner adapter's outstanding-run count spiked?). Remediation: temporarily raise the class-A rate-limit share; if the runner is the bottleneck, dispatch to a different runner (adapters are hot-swappable).
- **`online_loop_starvation`.** The mod-107 sampler's throughput dropped below the target sample rate. Owner: eval team. First step: check the class-C queue's drop rate and the vendor rate-limit share. Remediation: cut the class-A rate-limit share (usually a CI-burst overloaded the envelope); if sustained, raise the envelope with the vendor.
- **`quota_hard_stop`.** A cost centre hit the hard cap. Owner: cost-centre owner (not the eval team). First step: the cost-centre owner reviews the past week's runs, either raises the quota (finance approval), issues an override token (24 h escape), or lets the online loop resume when the month resets. The eval team's on-call is *not* paged for this.
- **`pricing_snapshot_drift`.** The invoice reconciliation script flagged > 5 % drift between the plane's roll-up and the vendor's invoice. Owner: eval team. First step: check for a mid-month price change the plane missed; check for a `pricing_snapshot` row that points at a retired vendor snapshot. Remediation: patch the snapshot; re-cost the affected runs; publish a correction to finance.

Every runbook lives in the same repo as the plane's code. Chapter 06 elevates the alerts to SLIs and error budgets.

## Summary

- The plane serves two triggers: **CI** (latency-bound, bursty, bounded per PR) and **online-eval** (cost-bound, sustained, throughput-critical). Treat them as first-class; give them separate priority classes.
- **Three priority classes** — A (CI gates), B (nightlies, back-fill), C (online loop) — with **weighted-fair scheduling** against a plane-wide **rate-limit envelope** (70 – 80 % of the vendor's stated cap).
- **Retries** use exponential backoff with jitter, capped at three retries and 60 s; every retry counts against the cost centre. **Non-retriable** failures fail the run cleanly.
- **Idempotency** is derived from the trigger context (`pr:...:commit:...:eval:...`), not from a UUID. Same key = same run (until the class's TTL).
- **Cost accounting** is per-row (`judgement.cost_usd`), per-run (`eval_runs.cost_usd`), per-team (materialised roll-up). Every cost row carries a `pricing_snapshot_id`; a monthly script reconciles against the vendor invoice.
- **Quotas** are enforced at **80 % soft** (warning) and **100 % hard** (HTTP 402). **Override tokens** and **emergency lifts** exist as time-boxed, audited escape hatches. The default is fail-closed.
- CI reads two artefacts: **`gate_result.json`** (pass / fail per the mod-106 gate rule, computed in the plane) and **`run_manifest.json`** (from chapter 04). Both live at stable per-run URLs.
- The **mod-107 online loop's scored-row store is a projection** of `eval_results` with `source='online_loop'` — one table, one schema, one query surface for every downstream dashboard.
- Every failure mode ships a runbook: `ci_gate_timeout`, `online_loop_starvation`, `quota_hard_stop`, `pricing_snapshot_drift`. Chapter 06 promotes them to SLIs.

Chapter 06 instruments the plane itself — four platform SLIs, error budgets, and the operability contract that turns "the plane the eval team wrote" into "the platform the eval team operates."
