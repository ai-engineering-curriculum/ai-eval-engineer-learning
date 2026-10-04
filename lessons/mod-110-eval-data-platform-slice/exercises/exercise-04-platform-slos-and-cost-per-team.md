# exercise-04: Platform SLOs and Cost per Team

**Estimated effort:** 3 hours

## Objective

Instrument the plane you shipped in exercises 01 – 03 as a **production system with public SLOs**. Wire the **four platform SLIs** from chapter 06 — eval-run success rate, latency by class, cost-per-team drift, queue-drop rate — publish the SLOs at a stable URL product teams can plan against, implement the error-budget burn-rate alerts, build the platform's own dashboard (four SLI tiles + runner-health + cost-centre roll-up + reproducibility-check + pricing-snapshot age + test-case age), and author the two runbooks for the alerts most likely to fire.

By the end of this exercise, the eval team can look at one dashboard and know whether the plane is healthy; a product team can read the SLO page before deciding to depend on the plane; and the on-call has a specific first-step runbook for the two most-common alerts. This is what turns "the plane the eval team wrote" (exercises 01 – 03) into "the platform the eval team operates."

This exercise reads from exercises 01 – 03 and from the mod-107 scored-row store (which chapter 05 projects onto `eval_results` with `source='online_loop'`). It is the last exercise that is strictly "inside the plane looking at the plane"; exercise-05 wires the plane to the two external triggers (CI, online loop).

## Prerequisites

- Chapters 01 and 06 of this module. Chapter 05 if the priority-class / rate-limit / cost-accounting vocabulary is not yet familiar.
- The schema from **exercise-01** (`eval_runs`, `eval_results`, `judgements`, `pricing_snapshots`), the test-case surface from **exercise-02** (`eval_sets`, `eval_cases`, `eval_set_current`), and the plane + three adapters from **exercise-03**. The dispatcher's `eval_runs.status` transitions, the adapters' `cost_report` output, and the exercise-01 `reproducibility_check` all feed this exercise's SLIs.
- A metrics backend. Prometheus is the shortest path (`prometheus_client` Python library + a local Prometheus server + Grafana for the dashboard). The exercise is portable to Datadog, Honeycomb, Chronosphere, or the org's homegrown; the shape of the SLIs does not change.
- An alerting surface — PagerDuty, Opsgenie, Slack webhooks, or the org's homegrown. A stub `alert_sink.py` that writes to a file is legal for local iteration but staging must use the real surface.
- A static-site host for the SLO page — GitHub Pages, S3 + CloudFront, Vercel, or the org's internal docs host. The URL must be stable across releases.
- Python 3.11+; `prometheus_client`, `fastapi` (for the SLO page API), `jinja2`, a Postgres client, `grafana-api` (optional, for provisioning dashboards from code).
- `scipy` for a one-time error-budget-math sanity check (burn-rate formulas from the Google SRE workbook).

## Set-up

1. Create `eval/platform/slo/`:

   ```
   eval/platform/slo/
   ├── config.yaml
   ├── slos/
   │   ├── success_rate.yaml
   │   ├── latency_by_class.yaml
   │   ├── cost_drift.yaml
   │   └── queue_drop.yaml
   ├── sli/
   │   ├── success_rate.py                # computes & exports metrics
   │   ├── latency_by_class.py
   │   ├── cost_drift.py
   │   ├── queue_drop.py
   │   └── auxiliary/
   │       ├── runner_health.py
   │       ├── reproducibility_sample.py  # scheduled check from exercise-01
   │       ├── pricing_snapshot_age.py
   │       └── test_case_age.py
   ├── burn/
   │   ├── burn_rate.py                   # fast / medium / slow burn alert math
   │   └── policy.py                      # error-budget policy enforcer (change-freeze gate)
   ├── alerting/
   │   ├── rules.yaml                     # Prometheus-style rules
   │   └── routing.yaml                   # per-SLI owner + severity map
   ├── dashboard/
   │   ├── grafana_dashboard.json
   │   ├── provision.py                   # ships the dashboard from code
   │   └── screenshots/                   # saved screenshots for the PR
   ├── slo_page/
   │   ├── app.py                         # FastAPI /slos + /slos/{id} + status page
   │   ├── templates/                     # SLI tiles, trend lines, error-budget gauges
   │   └── static_build.py                # pre-renders to a stable URL
   ├── runbooks/
   │   ├── latency_blown.md               # SLI #2 — latency by class
   │   ├── success_rate_burn.md           # SLI #1 — success rate
   │   ├── cost_drift.md                  # SLI #3 — cost drift
   │   └── queue_drop.md                  # SLI #4 — queue drops
   ├── capacity/
   │   └── model.py                       # chapter 06 capacity model: envelope × adapters × weights → sustainable RPS
   ├── tests/
   │   ├── test_sli_queries.py
   │   ├── test_burn_rate.py
   │   ├── test_policy.py
   │   ├── test_capacity_model.py
   │   └── test_alert_routing.py
   └── README.md
   ```

2. Fill `config.yaml`:

   ```yaml
   db:
     dsn: postgresql://eval:eval@localhost:5432/eval_platform
   metrics:
     backend: prometheus
     bind: 0.0.0.0:9091
   alerting:
     backend: pagerduty                   # or slack | opsgenie | file
     routing_config: ./alerting/routing.yaml
   dashboard:
     backend: grafana
     url: http://localhost:3000
     folder: eval-platform
   slo_page:
     bind: 0.0.0.0:8081
     base_url: https://eval.example.com/slos
   capacity:
     vendor_rps_envelope:
       openai: 500                        # per minute, 70% of stated vendor cap
       anthropic: 300
     adapter_concurrency:
       custom: 32
       promptfoo: 8
       braintrust: 16
     class_weights:
       class_a: 0.50
       class_b: 0.30
       class_c: 0.20
   error_budget_policy:
     healthy_remaining_pct: 25
     low_remaining_pct: 5
     spent_remaining_pct: 0
     freeze_hint_lifetime_hours: 168      # one week after SLI recovery
   ```

3. Author the four SLO documents under `slos/`. Each is a short YAML the SLO page renders:

   ```yaml
   # slos/success_rate.yaml
   id: success_rate
   name: Eval-run success rate
   sli: >
     Fraction of eval_runs that terminated in status='succeeded', excluding
     'throttled' and 'cancelled', per priority class, 7-day rolling window.
   target: 99.5
   per_class_targets:
     class_a: 99.5
     class_b: 99.0
     class_c: 99.0
   stratified_by: [priority]
   owner: eval-team
   runbook: runbooks/success_rate_burn.md
   error_budget:
     window_days: 7
     burn_alerts:
       fast:   { budget_fraction: 0.02, window_minutes: 60,   severity: page   }
       medium: { budget_fraction: 0.05, window_hours:   6,   severity: page    }
       slow:   { budget_fraction: 0.10, window_days:    3,   severity: ticket  }
   ```

   Repeat the shape for `latency_by_class.yaml`, `cost_drift.yaml`, `queue_drop.yaml`. The chapter 06 table has the per-SLI numbers.

4. Pre-populate historical data if your plane has not yet accumulated 7 days of runs. The acceptance criteria read against *at least* 48 hours of data — fire enough synthetic runs through exercise-03's three adapters, varying `priority`, `cost_centre`, `status` (succeed most, fail a few, throttle a few, cancel a few), to populate every SLI's numerator and denominator.

## Requirements

Produce a PR against your working branch that adds:

1. **Four SLI computations (`sli/*.py`)** — each exposes a Prometheus metric (or the equivalent in your backend) plus the raw SQL that computes the number from exercise-01's tables:
   - **`success_rate.py`** — class-stratified 7-day rolling success rate. Excludes `throttled` and `cancelled`. Emits `eval_run_success_rate{class="A|B|C"}`. Query is one `GROUP BY priority` over `eval_runs`.
   - **`latency_by_class.py`** — class-stratified rolling p95 latency, measured as `ended_at - triggered_at` (queue-wait counts). Emits a histogram per class. Class-A normalises against the canonical 500-case run as chapter 06 defines; class-C normalises per row.
   - **`cost_drift.py`** — per-cost-centre w-o-w cost delta. Reads `eval_runs.cost_usd` grouped by `cost_centre` × week. Emits `eval_cost_wow_delta{cost_centre="..."}`. Normalises for declared scale events (chapter 06: a team that opts into a `pricing_snapshot` change is excluded for the week).
   - **`queue_drop.py`** — class-stratified drop rate per 15-minute window. A dropped run is one the plane refused to enqueue — reads a `dropped_runs` table (migration below) or an in-memory counter the dispatcher increments.
2. **Auxiliary SLI tiles (`sli/auxiliary/*.py`)** — the five tiles chapter 06 shows below the four core SLIs:
   - **`runner_health.py`** — per-adapter success rate, mean latency, cost per row, running cost-per-row trend over 24h. The exercise-03 adapters feed this directly.
   - **`reproducibility_sample.py`** — a scheduled job that runs exercise-01's `reproducibility_check` against a daily sample (≤ 1 % of runs) and emits `eval_reproducibility_pass_rate`. Green = store trustworthy; red = recent runs may not reproduce.
   - **`pricing_snapshot_age.py`** — the age of the current `pricing_snapshot` per vendor; a snapshot > 30 days old emits yellow.
   - **`test_case_age.py`** — per suite, days since the last new case was added. A suite with 90+ days of no additions is yellow (chapter 03's "rot-check flow probably stopped" signal).
3. **Burn-rate alerts (`burn/*`)** —
   - `burn_rate.py` implements the Google SRE workbook burn-rate math: given an SLO target, an error budget, and a time window, compute the burn-rate and classify as fast / medium / slow. Unit tests cover the chapter 06 thresholds exactly.
   - `policy.py` implements the error-budget policy enforcer. On every SLI update, computes remaining budget per the per-SLI window; writes a `change_freeze_hint` flag that the plane's own CI reads (a plane PR cannot merge when the plane's own SLIs are "budget spent" — this is the dog-fooding of chapter 06).
4. **Alerting (`alerting/*`)** —
   - `rules.yaml` — one Prometheus-style rule per burn window per SLI. Every rule carries the `runbook_url` and the `owner`.
   - `routing.yaml` — per-SLI owner mapping. SLI #3 cost drift routes primary to the cost-centre owner (not the eval team); SLI #4 class-A drops route to the eval-team primary on-call; SLI #1 and SLI #2 route to the eval team.
   - Every page carries the runbook URL in the alert body; the runbook URL is clickable from PagerDuty / Slack.
5. **Dashboard (`dashboard/*`)** — one Grafana dashboard JSON, provisioned from code. Layout:
   - Top row — four tiles, one per SLI. Each tile shows: current value, SLO target, 7-day trend sparkline, error-budget-remaining gauge, link to runbook.
   - Second row — the five auxiliary tiles (runner health, reproducibility check, cost-centre roll-up top-20, pricing-snapshot age, test-case age).
   - Third row — a filterable table of the last 50 alerts with runbook link + owner + status.
   - A `provision.py` script that takes the JSON and uploads it; `screenshots/` has the dashboard captured for the PR body.
6. **Public SLO page (`slo_page/*`)** —
   - FastAPI `/slos` lists the four SLOs with name, target, current value, link to runbook, link to error-budget history.
   - `/slos/{id}` renders one SLO's details — SLI definition, target, error-budget shape, burn alerts, owner, runbook, last 90 days of SLI values.
   - A static site build (`static_build.py`) pre-renders the pages for the public URL in `config.yaml`.
   - The page is linked from the `README.md` of the plane repo so product teams discover it.
7. **Capacity model (`capacity/model.py`)** — chapter 06 arithmetic:
   - Inputs: current vendor envelope (RPS at 70 % utilisation), current adapter concurrency ceilings, current class weights.
   - Output: sustainable RPS per class at 70 % utilisation.
   - Enforces chapter 06's two rules — sum of per-class targets does not exceed plane total; a new team joining is a capacity decision (expose a `project_add_team(team, projected_class_c_rps)` function that returns pass / fail against current headroom).
   - Writes the current sustainable-RPS numbers to a Prometheus gauge the dashboard reads.
8. **Two runbooks (acceptance target) + two more (stretch)** —
   - **`runbooks/latency_blown.md`** — SLI #2. First step: `which class is blowing target?` Then: `is the queue backed up (scheduler) or is the runner slow (adapter / vendor)?` Remediation branches: shift rate-limit share across classes; escalate the vendor; spill to a secondary runner. Includes the exact SQL / PromQL to answer each diagnostic question.
   - **`runbooks/success_rate_burn.md`** — SLI #1. First step: `which class is burning?` Then: `which runner?` Then: `which failure classification (retriable-vendor / retriable-runner / non-retriable-schema / non-retriable-target)?` Remediation branches by cluster. Includes the SQL that groups failures by `(runner, status_reason)`.
   - Stretch includes `cost_drift.md` and `queue_drop.md`; the acceptance target is the two most-common-to-fire runbooks.
9. **Dog-food gate (`burn/policy.py` + plane CI integration)** — a merge-time check in the plane's own CI: a PR against the plane repo cannot merge when the SLI dashboard is in "budget spent" for the SLI the PR touches. This is the chapter 06 error-budget-policy mechanic made executable. Overridable via a labelled bypass that is logged + reviewed weekly.
10. **Postmortem template (`runbooks/POSTMORTEM_TEMPLATE.md`)** — the chapter 06 "every incident produces a postmortem" artefact. Timeline, root cause, action items with owner + due date, follow-up review two weeks later. One worked example (synthetic — simulate a 2-hour latency blow and write the postmortem against it).
11. **Tests (`tests/*`)** — unit tests for each SLI query (hand-authored fixtures with known-answer expected outputs), the burn-rate math (chapter 06 numbers exactly), the policy enforcer (a budget-spent flag freezes the plane PR; an override unfreezes with an audit row), the capacity model (adding a team that overflows the envelope is refused), and the alert routing (SLI #3 does not page the eval team by default).
12. **README (`README.md`)** — three sections:
    - **For a product team.** Here are the four SLOs; here is the stable URL; here is how to depend on the plane with confidence; here is what "change freeze" looks like when it fires.
    - **For the on-call.** Here is the dashboard; here are the four runbooks; here is the weekly SLI review ritual; here is the postmortem template.
    - **For the eval-team lead.** Here is the quarterly SLO review check (chapter 07's four questions); here is the capacity-model `project_add_team` call; here is the dog-food gate's override process.

## Starter guidance

- **Start from the chapter 06 table.** Every SLI's numbers are already pinned in the chapter — the exercise's job is to make them computable, dashboardable, and alert-able. Do not re-derive the targets; implement them.
- **The burn-rate math is small; the policy is where the care goes.** The burn-rate formulas take 20 lines; the policy enforcer that freezes plane PRs on a spent budget is where you spend the design care. Dog-fooding the policy is the single move that makes the SLOs real inside the eval team.
- **The reproducibility-check tile is not optional.** Chapter 06 calls it out specifically — it is the SLI that says the exercise-01 schema is doing what it promised. Wire it with the same care as the four core SLIs even though it is a "tile below."
- **Error budgets are decisions, not hopes.** Every SLO document names what happens at healthy / low / spent. Product teams reading the SLO page see the policy, not just the number. The transparency is what makes the SLI credible.
- **Alert routing: SLI #3 does not page the eval team by default.** Cost drift is primarily a cost-centre-owner conversation. If the eval team is paged on every cost drift, they become finance's shadow team, and the real signal (SLI #1 success rate) gets lost. Route by chapter 06's table.
- **Runbooks are scripts the on-call runs.** First line is a discriminating question with the SQL to answer it. Not "the eval team investigates." If a runbook does not have the exact query the on-call will type, it is prose, not a runbook.
- **Capacity model is small arithmetic, not a simulation.** `envelope × class_weight × adapter_concurrency × utilisation_target → sustainable RPS` is three multiplies and a cap. Resist the temptation to build a discrete-event simulator; the three-multiply model is what the eval-team lead will actually use.
- **Publish the SLO page at a stable URL on day one.** Even if the dashboard is scrappy, the URL is the contract. Product teams linking to a URL that moves is worse than a plain `<pre>` of the four numbers at the right URL.
- **The dog-food gate will annoy you.** The first time a plane PR gets blocked by the plane's own SLIs you will want to disable the gate. Do not — the whole chapter 06 argument is that the gate makes the SLOs real for the people who set them.
- **One postmortem beats a dozen alert rules.** The chapter 06 "every incident produces a postmortem" is not a documentation task; it is the mechanism that turns alerts into tuned alerts. Shipping one real postmortem (even synthesised for the exercise) is worth more than four pre-written runbooks.

## Acceptance criteria

You are done when:

- The four SLIs compute against the exercise-01 schema and emit Prometheus (or equivalent) metrics. The queries' outputs match hand-computed expected numbers on the fixture data to 3 decimals.
- The dashboard renders the four SLI tiles + the five auxiliary tiles in Grafana (or equivalent); screenshots are in the PR.
- The public SLO page is deployed at the stable URL in `config.yaml`; the four SLOs are visible with current value, target, trend, error budget remaining.
- The burn-rate math is exercised by unit tests against the chapter 06 numbers verbatim (fast = 2 % in 1 h; medium = 5 % in 6 h; slow = 10 % in 3 d for SLI #1).
- An alert fires on a synthesised event: fail 10 % of class-A runs over 15 minutes → the fast-burn page fires → the alert body carries the `success_rate_burn.md` runbook URL.
- The dog-food gate blocks a plane PR when the dashboard shows "budget spent" for the SLI the PR touches; an override requires an explicit label and lands an audit row.
- The capacity model's `project_add_team(team, 15_rps)` returns `pass` when there is headroom and `fail` with a diagnostic when there is not.
- The two acceptance-target runbooks are legible by someone who has not read chapter 06 — hand them to a colleague and confirm they could execute the first step without additional context.
- The reproducibility-sample tile runs on a daily schedule and reports a pass rate; a known-drifted fixture from exercise-01 flips the tile to red.
- SLI #3 (cost drift) alerts route primary to the cost-centre owner; the eval team is secondary at the > 50 % threshold.
- Tests pass in CI.
- The three audiences in `README.md` are complete and read naturally without reference to chapter 06.

## Stretch goals

- **The other two runbooks.** `cost_drift.md` and `queue_drop.md`, with the same discipline — discriminating question + SQL + remediation branches.
- **Multi-window burn alerts.** The SRE workbook recommends combining short + long burn-rate windows per severity (the "sliding window" pattern) to reduce false pages. Replace the single-window alerts with the two-window variant and compare false-positive rates on a replay of 30 days of synthetic data.
- **Noise-tuning the alert rules.** After one week in staging, review every alert that fired. For every alert that fired without an action, either tune the threshold or delete the alert. Document the deltas in `alerting/CHANGELOG.md`.
- **Weekly SLI review automation.** A scheduled job emits a weekly digest (Slack post or email) with: the four SLIs' trend, every alert that fired and its outcome, the current error budget per SLI, the dog-food gate's firing count, and the top three postmortem action items from the past month.
- **Per-adapter SLO carve-out.** Give each runner adapter its own SLI contribution (runner-level success-rate, runner-level cost per row). When a vendor degrades, the dashboard surfaces which runner is dragging the plane's number; the plane's own SLO is unaffected if traffic can be shifted.
- **Capacity-model forecasting.** Add a 90-day forecast of class-C RPS from mod-107 online-loop growth. Alerts fire when the forecast predicts hitting the envelope within 60 days — the chapter 06 "escalate before you need to" signal.
- **Status-page integration.** Push the SLO page onto the org's status-page vendor (Statuspage, Instatus) so product teams see the plane's health alongside the rest of the stack. The `incidents` feed on the status page is derived from the postmortem archive.
- **A real postmortem.** Replace the worked synthetic example with a real incident when one happens. Follow chapter 06's two-week follow-up review discipline.

## What this exercise does *not* cover

You are not building the CI gate's `gate_result.json` or the mod-107 online-loop integration (both exercise-05); you are not building the escalation ticket to `model-evaluation-engineer` (chapter 07 — not an exercise); you are not re-deriving the lineage keys or the test-case contribution flow (exercises 01 – 02). You are shipping the four public SLOs and the operability apparatus that turns the plane into a platform.
