# exercise-05: CI and Online-Eval Integration

**Estimated effort:** 4 hours

## Objective

Wire the plane (exercises 01 – 03) into its **two trigger surfaces**:

- The **mod-106 PR / deploy gate** — bursty, latency-bound, every call rate-limited, retried with idempotency, billed to a cost centre against a quota, returning `gate_result.json` + `run_manifest.json`.
- The **mod-107 sampled-trace online loop** — sustained, cost-bound, class-C runs that write back into `eval_results` with `source='online_loop'` so the mod-107 drift monitor queries the same table every other dashboard reads.

By the end you have a plane that CI calls and that the online loop fires at, with a **weighted-fair scheduler** across three priority classes, a **plane-wide rate-limit envelope** against the vendor, **exponential-backoff retries** with jitter, **idempotency keys** derived from the trigger context (not UUIDs), **per-team cost accounting** that reconciles against the vendor invoice, **80 % soft / 100 % hard quotas** with time-boxed override tokens, and the **two artefacts** (`gate_result.json`, `run_manifest.json`) that CI reads at stable per-run URLs.

This is the last exercise. It makes the plane *useful* — before this exercise the plane runs when you type `plane run`, after it the plane is what CI calls and what the online loop talks to. Every chapter 06 SLI you wired in exercise-04 lights up for real traffic.

## Prerequisites

- Chapter 05 of this module. Chapter 06 if the plane's own SLI shape is not yet in muscle memory.
- The schema (**exercise-01**), the test-case surface (**exercise-02**), the plane + three adapters (**exercise-03**), the SLO dashboard (**exercise-04**). The CI gate reads the `eval_set_hash` from exercise-02's `eval_set_current`; the dispatcher is the one from exercise-03; the dashboard shown here lights up with real traffic.
- A CI system — GitHub Actions, GitLab CI, Buildkite, or Jenkins. The reference implementation below targets GitHub Actions; the shape ports directly.
- A deployable mod-106 target. Reuse the mod-106 PR-gate workflow from that module's exercise-01 if it exists, or ship a minimal mock PR workflow in this exercise's `.github/` directory.
- A mod-107 online-loop sampler — the one from mod-107 exercise-01, or a stub that emits ~ 100 synthetic traces per hour against a fixed sample bucket. Either is fine; the plane's integration contract does not depend on the sampler's shape.
- A message bus (Redis Streams, RabbitMQ, SQS, or Kafka) for the weighted-fair scheduler. Redis is the shortest path — one container, `redis-py`, a 30-line token-bucket. The exercise supports in-process fan-out as a fallback for local iteration but the weighted-fair semantics must be exercised through the queue.
- Python 3.11+; `fastapi` or `litestar` (the plane's control API), `redis` (if you pick Redis), `prometheus_client` (SLI emission continues from exercise-04), `httpx` (CI → plane calls).
- The vendor model keys from exercise-03. The envelope defaults to 70 % of the vendor's stated per-minute cap; look up the vendor cap in their dashboard before wiring the envelope.

## Set-up

1. Create `eval/platform/integration/`:

   ```
   eval/platform/integration/
   ├── config.yaml
   ├── control_api/
   │   ├── app.py                        # FastAPI; POST /v1/runs, GET /v1/runs/{id}
   │   ├── routes.py
   │   ├── auth.py                        # mTLS or OIDC; dev shim is per-config
   │   └── middleware_idempotency.py
   ├── scheduler/
   │   ├── token_bucket.py                # per-class token bucket
   │   ├── weighted_fair.py               # scheduler loop
   │   ├── envelope.py                    # plane-wide vendor rate envelope
   │   └── drops.py                       # capacity / envelope / quota drop recorder
   ├── retry/
   │   ├── classifier.py                  # retriable-vendor | retriable-runner | non-retriable
   │   ├── backoff.py                     # exponential + full jitter
   │   └── idempotency_key.py             # chapter 05 derivation rules
   ├── cost/
   │   ├── accounting.py                  # per-row, per-run, per-team roll-ups
   │   ├── pricing_snapshot.py            # effective_from / effective_to pointer resolver
   │   ├── invoice_reconcile.py           # monthly reconciliation script
   │   └── quota.py                       # 80% / 100% enforcement + override tokens
   ├── artefacts/
   │   ├── gate_result.py                 # emits gate_result.json
   │   ├── run_manifest.py                # emits run_manifest.json (consumes exercise-03's)
   │   └── artefact_store.py              # stable per-run URL + object-store write
   ├── ci/
   │   ├── github_action.yaml             # the mod-106 PR gate's call into the plane
   │   ├── submit.py                      # CLI the gate invokes
   │   └── render_comment.py              # PR-comment template reading gate_result.json
   ├── online_loop/
   │   ├── adapter.py                     # mod-107 sampler → plane class-C submission
   │   └── fire_and_forget.py             # no polling; results land async
   ├── migrations/
   │   ├── 0005_dropped_runs.sql
   │   ├── 0006_quota_overrides.sql
   │   └── 0007_pricing_snapshots_history.sql
   ├── runbooks/
   │   ├── ci_gate_timeout.md
   │   ├── online_loop_starvation.md
   │   ├── quota_hard_stop.md
   │   └── pricing_snapshot_drift.md
   ├── tests/
   │   ├── test_idempotency.py
   │   ├── test_weighted_fair.py
   │   ├── test_envelope.py
   │   ├── test_retry_classifier.py
   │   ├── test_quota.py
   │   ├── test_pricing_snapshot.py
   │   ├── test_invoice_reconcile.py
   │   ├── test_gate_result.py
   │   └── test_online_loop_fire_and_forget.py
   └── README.md
   ```

2. Fill `config.yaml`:

   ```yaml
   db:
     dsn: postgresql://eval:eval@localhost:5432/eval_platform
   api:
     bind: 0.0.0.0:8080
     auth: oidc                           # or mtls, local_dev (dev only)
   scheduler:
     backend: redis
     url: redis://localhost:6379
     classes:
       class_a: { share: 0.50, rate_limit_rpm: 250 }
       class_b: { share: 0.30, rate_limit_rpm: 150 }
       class_c: { share: 0.20, rate_limit_rpm: 100 }
     envelope:
       openai: { rpm: 500, provider_cap_rpm: 715 }   # 70% of 715
       anthropic: { rpm: 300, provider_cap_rpm: 430 }
   retry:
     max_retries: 3
     max_backoff_s: 60
     jitter: full
   idempotency:
     ttl_by_class:
       class_a: 24h
       class_b: 168h
       class_c: inf
   quotas:
     warn_at_pct: 80
     hard_at_pct: 100
     override_token_ttl_h: 24
   gate_result:
     mod106_threshold_source: ../../mod-106-eval-gated-cicd/thresholds/
   artefact_store:
     backend: s3                          # or local-fs, gcs
     bucket: eval-platform-artefacts
     url_pattern: https://eval.example.com/runs/{run_id}/{artefact}.json
   ```

3. Apply the three migrations:
   - `0005_dropped_runs.sql` — a `dropped_runs(dropped_id, submitted_at, dropped_at, cost_centre, priority, reason)` table. Rows written when the plane refuses to enqueue (capacity / envelope / quota).
   - `0006_quota_overrides.sql` — a `quota_overrides(token, cost_centre, added_cap_usd, issued_by, issued_at, expires_at, reason, uses_remaining)` table.
   - `0007_pricing_snapshots_history.sql` — adds `effective_from`, `effective_to`, `superseded_by` to the exercise-01 `pricing_snapshots` table (if not already there).

4. Register at least two cost centres with distinct quotas. Example:

   ```sql
   INSERT INTO cost_centres (id, owner, monthly_quota_usd) VALUES
     ('team-support/project-bot', 'alice@example.com', 2000.00),
     ('team-safety/project-redteam', 'bob@example.com', 500.00);
   ```

5. Point the mod-106 PR workflow at `POST /v1/runs` (or install a minimal mock workflow if mod-106 is not deployed). Point the mod-107 sampler at the online-loop adapter.

## Requirements

Produce a PR against your working branch that adds:

1. **Control API (`control_api/*`)** —
   - `POST /v1/runs` accepts an `EvalRunSpec` body plus the headers `X-Cost-Centre`, `X-Trigger`, `X-Idempotency`, `X-Priority`, optional `X-Quota-Override`.
   - Rejects a request with HTTP 400 if the spec is malformed, HTTP 402 if the cost centre is over hard-cap and no override is present, HTTP 429 if the plane is temporarily refusing capacity (class-C-dropped), HTTP 503 if the plane itself is down (circuit-broken).
   - Returns HTTP 201 with the `run_id` immediately; subsequent `GET /v1/runs/{run_id}` returns the current `status` + manifest URLs.
   - Idempotency middleware (`middleware_idempotency.py`) resolves `X-Idempotency` against the existing-run table per chapter 05's rules — same key in progress → return existing `run_id`; same key terminal within TTL → return existing manifest; same key terminal past TTL → fresh run; missing key → reject with HTTP 400.
2. **Weighted-fair scheduler (`scheduler/*`)** —
   - `token_bucket.py` implements a per-class token bucket with the configured RPM.
   - `weighted_fair.py` is the dispatcher loop — pulls from the three class queues at the configured share (50 % / 30 % / 20 %); a class-A burst does not starve class-C and vice versa.
   - `envelope.py` is the plane-wide envelope — the sum of outstanding dispatches to a vendor cannot exceed `envelope.rpm`. Enforced *before* calling the runner adapter (chapter 05's "runner adapter never sees a 429").
   - `drops.py` writes to `dropped_runs` whenever the plane refuses to enqueue; the chapter 06 SLI #4 reads this table.
3. **Retries (`retry/*`)** —
   - `classifier.py` classifies every runner / vendor error into retriable-vendor (429 / 5xx / TCP reset), retriable-runner (subprocess crash, parsing error), or non-retriable (judge useless response, schema violation, target-app bug).
   - `backoff.py` implements exponential with full jitter capped at 60 s, max 3 retries. Every retry is a new row in a `run_attempts` table (or a column on `eval_runs` with the attempt count); the cost accounting counts every attempt against the cost centre.
   - A non-retriable failure terminates the run cleanly with the specific failure classification on `eval_runs.status_reason`.
4. **Idempotency key derivation (`retry/idempotency_key.py`)** — chapter 05's exact derivation rules:
   - `pr_gate` — `pr:<pr_id>:commit:<sha>:eval:<eval_set_id>:rubric:<rubric_hash>`
   - `deploy_gate` — `deploy:<deploy_id>:eval:<eval_set_id>:rubric:<rubric_hash>`
   - `online_loop` — `online:<sample_bucket>:trace:<trace_id>:rubric:<rubric_hash>`
   - `scheduled` — `schedule:<cron_id>:window:<yyyy_mm_dd>:eval:<eval_set_id>:rubric:<rubric_hash>`
   - `manual` — `user:<uid>:spec_hash:<spec_hash>:ts:<yyyy_mm_dd_hh_mm>`

   A unit test enforces that two triggers with the same context produce the same key, and that a random-UUID key is rejected.
5. **Cost accounting (`cost/*`)** —
   - `accounting.py` writes per-row cost at `EvalResultRow` ingest time (`judgement.cost_usd` from token count × `pricing_snapshot` × unit price); rolls per-run on run termination; the per-team view is a materialised view (or nightly job) grouped by `(cost_centre, week)`.
   - `pricing_snapshot.py` resolves the effective snapshot per `eval_runs.started_at`. A monthly price change inserts a new snapshot with `effective_from = now()` and updates the prior snapshot's `effective_to`; historic runs read their snapshot pointer unchanged.
   - `invoice_reconcile.py` runs monthly; joins the per-team roll-up against the vendor invoice line items; flags discrepancies > 5 % with the probable cause (missed `pricing_snapshot` change, un-attributed runs). Emits an alert routed to `pricing_snapshot_drift.md`.
   - `quota.py` enforces the chapter 05 rule — 80 % soft (header warning on the response), 100 % hard (HTTP 402). Override tokens are time-boxed with the TTL in config; each use burns one of `uses_remaining` and is audited.
6. **Gate artefacts (`artefacts/*`)** —
   - `gate_result.py` reads the exercise-02 `eval_set_hash` the spec names, reads the mod-106 threshold config at the gate-config path, computes pass / fail per rubric per cohort, writes `gate_result.json` with the chapter 05 shape (metrics array, cost centre, run cost, month-to-date and quota).
   - `run_manifest.py` consumes the exercise-03 manifest and merges the plane's fields (`cost_centre`, `cost_centre_month_to_date_usd`, `cost_centre_month_quota_usd`, `attempts`, `cache_hit_count`).
   - `artefact_store.py` writes both to a stable per-run URL (`https://eval.example.com/runs/{run_id}/gate_result.json` + `.../run_manifest.json`) and to an object-store path. The URL does not change for a run's retention window; changing it is a schema migration.
7. **CI integration (`ci/*`)** —
   - `github_action.yaml` is the mod-106 PR gate's call into the plane: builds the `EvalRunSpec` from the PR context + `eval_set_current`, derives the idempotency key, POSTs to `/v1/runs`, polls `/v1/runs/{run_id}` until terminal, downloads `gate_result.json`, blocks the merge if the gate failed, posts a PR comment via `render_comment.py`.
   - `submit.py` is the CLI the workflow invokes. Supports a `--dry-run` that validates the spec without dispatching.
   - `render_comment.py` renders the gate's pass / fail per rubric, the cost delta, the cost centre's month-to-date vs quota, and clickable links to the three `artefact_uris` (eval_results query, vendor-native, OpenLineage).
8. **Online-loop integration (`online_loop/*`)** —
   - `adapter.py` is the sampler → plane bridge. For every sampled trace, builds a one-case `EvalRunSpec` with `priority: class_c`, derives the idempotency key, POSTs to `/v1/runs`. Fire-and-forget — no polling.
   - The sampler reads the plane's capacity model from exercise-04 and refuses to oversubscribe (`sample_rate <= class_c_sustainable_rps_at_70_pct`); a config change that would overflow is rejected at config-load time.
   - Verify the mod-107 drift monitor's query (`SELECT * FROM eval_results WHERE source='online_loop'`) returns the plane's class-C rows unchanged — chapter 05's "scored-row store is a projection" invariant.
9. **Four runbooks (`runbooks/*`)** — chapter 05's named runbooks:
   - **`ci_gate_timeout.md`** — the CI call did not return terminal before deadline. First step: queue depth (class-A backed up) vs runner adapter (outstanding-run spike). Remediation branches: temporarily raise class-A share; dispatch to a different runner.
   - **`online_loop_starvation.md`** — mod-107 sampler throughput dropped below target. First step: class-C queue drop rate vs vendor envelope share. Remediation: cut class-A share; raise envelope with vendor.
   - **`quota_hard_stop.md`** — a cost centre hit the hard cap. **Owner is the cost-centre owner, not the eval team.** First step: review the past week's runs; raise quota, issue override, or wait for month reset.
   - **`pricing_snapshot_drift.md`** — invoice reconciliation found > 5 % drift. First step: mid-month price change the plane missed vs un-attributed runs. Remediation: patch the snapshot; re-cost affected runs; publish finance correction.
10. **Tests (`tests/*`)** —
    - **Idempotency.** Same key in progress → same `run_id`; same key terminal within TTL → cached manifest; different key → distinct run. UUID → rejected.
    - **Weighted-fair.** A class-A burst does not starve class-C — simulated 50 class-A submissions + a steady class-C stream for 60 s; class-C throughput stays within 90 % of its target.
    - **Envelope.** The envelope cap holds under 2× over-submission — the plane refuses rather than letting the vendor 429.
    - **Retry classifier.** A 429 is retriable-vendor; a judge "I can't evaluate this" is non-retriable; a schema-violation is non-retriable; a subprocess crash is retriable-runner.
    - **Quota.** 80 % returns a warning header; 100 % returns 402; override token raises the cap by `added_cap_usd` and expires after `TTL`.
    - **Pricing snapshot.** A mid-month price change inserts a new snapshot; historic runs read the old price; the reconciliation script flags a > 5 % drift on fabricated invoice data.
    - **Invoice reconcile.** Correct per-team totals reconcile to within 1 % of a fabricated vendor invoice; a mis-priced snapshot produces the expected > 5 % flag.
    - **Gate result.** `gate_result.json` matches the chapter 05 shape; a failing rubric blocks the mock PR.
    - **Online loop fire-and-forget.** 100 class-C submissions complete without any poll; mod-107 drift query returns 100 rows with `source='online_loop'`.
11. **README (`README.md`)** — three sections:
    - **For CI engineers.** The `POST /v1/runs` contract; the GitHub Action; the PR comment shape; what each HTTP status means; how to safely retry; how to read the artefacts.
    - **For the online-loop operator.** The fire-and-forget contract; how to raise the sample rate safely (capacity model gate); how to confirm the drift monitor sees the plane's rows.
    - **For the on-call.** The four runbooks; how to issue a quota override; how to run the invoice reconciliation; the dashboard tiles that light up for each failure mode.

## Starter guidance

- **Derive the idempotency key from context, not UUID.** This is the chapter 05 move that makes CI retries safe. A UUID key defeats idempotency; a per-request UUID is the single most common way a CI retry doubles the cost.
- **Enforce the envelope *before* calling the runner adapter.** Chapter 05's whole retry story depends on the runner never seeing a vendor 429 under normal load. If the envelope is enforced inside the adapter, it is already too late — the plane cannot distinguish "we throttled" from "the vendor throttled."
- **Weighted-fair is small — do not reach for Kafka.** A per-class Redis token bucket is 60 lines; a per-class in-memory bucket is 20. If you have Kafka, use it; if you do not, do not install it for this exercise.
- **Fail-closed on quotas.** Chapter 05's rule — fail-closed with override tokens, not fail-open with warnings — is the one every eval team eventually learns the hard way. Ship the HTTP 402; add the override token on day one.
- **The override token is time-boxed and audited.** A 24-hour added-cap-of-$500 token is an escape hatch, not a quota raise. The eval team's lead reviews override uses weekly; the dashboard surfaces who issued what.
- **Reconciliation is a Python script, not a spreadsheet.** The chapter 05 "monthly, 5 % threshold" is executable or it is not real. Run it on dummy invoices for the exercise; the real one lands once the plane has a month of real traffic.
- **`gate_result.json` is derived from `eval_results` + the mod-106 threshold config.** The gate rule is in mod-106; the computation lives here. Do not duplicate the threshold config; point at the mod-106 path.
- **The online-loop adapter must be a one-liner wrapper over `POST /v1/runs`.** If the online loop needs special-casing in the plane, the plane's contract is wrong. The only thing the online-loop trigger does differently is set `X-Priority: class_c` and pick an idempotency key.
- **Mod-107's scored-row store is a projection, not a copy.** The query `SELECT * FROM eval_results WHERE source='online_loop'` *is* the mod-107 scored-row store. Do not build a second table; the chapter 05 invariant is the one query surface.
- **Dashboards start lighting up for real.** As CI traffic hits the plane, the exercise-04 SLI tiles stop being synthetic. Watch the first week carefully — tune alerts that fire without action.
- **Four runbooks, not forty.** Chapter 05 names four specific failure modes. Resist the temptation to pre-write runbooks for every possible failure; let the actual pages dictate what else gets a runbook.

## Acceptance criteria

You are done when:

- `POST /v1/runs` accepts an `EvalRunSpec` + headers and returns a `run_id`; `GET /v1/runs/{run_id}` returns status + manifest URLs.
- A GitHub Actions (or equivalent) PR workflow invokes the plane, polls to termination, downloads `gate_result.json`, blocks on failure, and posts a PR comment.
- Two triggers of the same PR + commit + eval + rubric produce the same `run_id` and the same manifest; a UUID idempotency key is rejected.
- A class-A burst of 50 requests leaves class-C's throughput within 90 % of its target over 60 s — the weighted-fair scheduler holds.
- A 2× over-submission against the vendor envelope results in `dropped_runs` entries, not vendor 429s observed in the runner adapter logs.
- A failing judge-model call is retried up to 3× with exponential backoff and full jitter; the cost accounting records every attempt against the cost centre.
- A cost centre reaching 80 % month-to-date receives an `X-Quota-Warning: 82%` response header; reaching 100 % receives HTTP 402 with the chapter 05 message; an override token raises the cap by the token's amount and expires on time.
- A `pricing_snapshot` change inserts a new snapshot with `effective_from = now()`; historic `eval_runs` read the old price; the monthly reconciliation flags a > 5 % drift on fabricated test invoice data.
- `gate_result.json` lives at a stable URL per run; the PR comment renders the pass / fail per rubric with the cost-centre spend and quota.
- The mod-107 online-loop adapter submits class-C runs fire-and-forget; `SELECT * FROM eval_results WHERE source='online_loop'` returns the plane's rows for the mod-107 drift query to consume unchanged.
- Four runbooks live under `runbooks/`; each names an owner, a first-step diagnostic question with the exact SQL or PromQL, and remediation branches.
- The exercise-04 SLI tiles update with real CI + online-loop traffic — SLI #1 success rate, SLI #2 latency by class, SLI #4 queue-drop rate all show non-synthetic values after one hour of load.
- Tests pass in CI; the three audiences in `README.md` are complete and read naturally.

## Stretch goals

- **Deploy-gate wiring.** In addition to the PR gate, wire the deploy-gate shape (`X-Trigger: deploy_gate`); test that a canary deploy triggers a class-A run against a canary-tagged eval set and blocks the roll-out on failure. Reuse the mod-106 deploy-gate workflow if available.
- **Progressive priority (class B back-fill).** Add a class-B workflow — a scheduled nightly re-score of the last 7 days' rot-check cases, enqueued at class B so it does not interfere with CI or the online loop. Confirm the weighted-fair scheduler holds under all three classes active.
- **Spill-to-secondary-runner on degradation.** Wire a policy where class-A dispatches shift from the primary runner to a secondary runner (one of exercise-03's three adapters) when the primary's success rate drops below 95 % for 5 minutes. The dashboard shows the spill event.
- **Per-tenant chargeback export.** Monthly, emit a per-cost-centre invoice (CSV + Markdown) with the roll-up, the pricing-snapshot history, the override-token audit, and the reconciliation delta against the vendor invoice. Hand it to finance to confirm the shape is usable.
- **Circuit breaker on the control API.** When the plane's own SLI dashboard shows SLI #1 success rate < 50 % for 5 minutes, the control API returns HTTP 503 on new submissions (not drops) so CI can distinguish "the plane is broken" from "my request was dropped." Chapter 06's error-budget-policy gate reads this too.
- **Override-token UI.** A small page in the exercise-02 contribution UI where a cost-centre owner mints override tokens; the audit page shows weekly use. The eval-team lead's quarterly review reads the audit.
- **Multi-region envelope.** If the vendor has per-region rate limits (OpenAI, Azure OpenAI, Vertex), the envelope is split per-region and class-C traffic routes to the region with headroom. Chapter 06's capacity-model forecast reads the per-region envelope.
- **Half-year postmortem review.** Once the plane has six months of real traffic, pull every alert and every postmortem from exercise-04's archive; produce a Markdown digest of the top 5 recurring failures and the chapter 05 escalations each would warrant. This is the artefact the chapter 07 delegation ticket cites.

## What this exercise does *not* cover

You are not re-shipping the schema, the test-case surface, the plane dispatcher, the three adapters, or the four SLI tiles — those are exercises 01 – 04. You are not writing the chapter 07 delegation ticket to `model-evaluation-engineer`. You are wiring the plane to its two trigger surfaces so every exercise before this one lights up for real traffic — the product-eval slice is operable, the chapter 06 SLIs run against real data, and the chapter 07 hand-over artefacts (schema, adapter interface, SLI history, test-case archive) now have a working production surface behind them.
