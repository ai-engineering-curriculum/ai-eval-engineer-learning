# Platform SLOs and Operability: Four SLIs, Error Budgets, Runbooks

## Motivation

The plane, at this point, works. Teams submit runs, the runners score them, the schema stores them, quotas hold, cost accounts reconcile. The plane is a *script that works*.

A script that works is not a platform. The moment a product team depends on the plane — the moment their PR gate blocks on `POST /v1/runs` returning cleanly, the moment their online-loop drift monitor is silent because the plane's throughput has quietly halved — the plane is a *production system*. Production systems are on-call.

The chapter's job is to fix the four platform-level SLIs the eval team is on-call for, define their error budgets, wire the SLI dashboard, and write the runbooks the on-call reads at 2 am. The SLIs are customer-facing commitments — the teams that use the plane see them and plan against them. The runbooks are executable — the "runbook is a script the on-call runs" rule from mod-106 chapter 03 applies to the plane's own operability.

Everything the chapter builds is inside the plane looking at the plane. The mod-107 dashboards look at the *application*; this dashboard looks at *the platform serving the eval program*. Both are load-bearing.

## Core concepts

### The four platform SLIs

Four indicators cover the plane's health. Each has an SLI (the measurement), an SLO (the target), an error budget (the room to fail), a customer-visible dashboard tile (green / yellow / red), and a runbook.

| # | SLI | SLO | Error budget shape | Owner |
|---|---|---|---|---|
| 1 | Eval-run success rate | ≥ 99.5 % of runs terminate in `succeeded` over rolling 7d, per priority class | 0.5 % of runs per week may fail non-retriably; burn faster than 4× → page | Eval team |
| 2 | Eval-run latency by class | p95 latency ≤ (class A: 5 min for a 500-case run; class B: 30 min; class C: 5 min per row) | Any class's p95 blows target for > 1 h → page; blows for < 1 h and self-heals → ticket | Eval team |
| 3 | Cost-per-team drift | Week-over-week cost delta per cost centre within ±20 % (unless the team's spec accepts the increase) | > 20 % w-o-w drift on a cost centre → email the cost-centre owner; > 50 % → page eval team | Cost-centre owner primary; eval team secondary |
| 4 | Queue-drop rate | ≤ 0.1 % of enqueued runs are dropped for capacity or quota reasons; drop-rate stratified by priority class | > 0.1 % class-C drops sustained for > 15 min → ticket; > 0.1 % class-A drops at all → page | Eval team |

Two rules:

- **The SLOs are customer-visible.** They live at a stable URL (`https://eval.example.com/slos` or the org's status-page equivalent). Product teams read them before choosing to depend on the plane; they plan their PR-gate deadlines against SLI #2, they plan their online-loop budget against SLI #4, they plan their monthly finance forecast against SLI #3.
- **The error budgets are decisions.** An error budget is not "how much we hope to spend on outages." It is the room the eval team is *authorised* to spend on risky changes (a new runner adapter, a schema migration, a vendor swap). When the budget is spent, non-essential change stops until the SLI recovers. This is the Google SRE convention (see the SRE workbook) applied to the plane.

The rest of the chapter walks each SLI.

### SLI #1 — Eval-run success rate

Measurement:

```
success_rate_7d[class] =
    count(eval_runs WHERE status='succeeded' AND priority=class AND ended_at > now() - 7d)
  / count(eval_runs WHERE status IN ('succeeded','failed','cancelled') AND priority=class AND ended_at > now() - 7d)
```

Notes:

- `throttled` runs (quota hard-cap) do **not** count against success rate — they are a legitimate refusal, not a platform failure. If a cost centre is throttled, that is a quota-owner conversation, not a plane bug.
- `cancelled` (the trigger explicitly cancelled) does not count against success rate for the same reason.
- Class-stratified — a spike of class-C failures during a vendor outage does not silently drag the class-A number below its target.

Error budget:

- Weekly budget = 0.5 % of runs. If the plane processed 20 000 runs last week, the budget is 100 runs of failures. A vendor outage that failed 30 runs in five minutes is *inside* the budget; a schema migration that fails 200 runs over an afternoon *is not*.
- Burn-rate alerts follow the SRE workbook pattern:
  - 2 % of weekly budget burned in 1 h → page (fast burn)
  - 5 % of weekly budget burned in 6 h → page (medium burn)
  - 10 % of weekly budget burned in 3 d → ticket (slow burn)

Runbook: `runbooks/success_rate_burn.md`. First step: which class is burning? Which runner? Which failure classification (retriable-vendor / retriable-runner / non-retriable-schema / non-retriable-target)? Remediation depends on the cluster.

### SLI #2 — Eval-run latency by class

Measurement:

```
p95_latency_1h[class] = percentile(0.95, ended_at - triggered_at
                                        WHERE priority=class AND ended_at > now() - 1h)
```

Notes:

- Latency is `ended_at - triggered_at`, *not* `ended_at - started_at`. Queue-wait counts. This is what CI cares about — the developer waited for the gate.
- Class-A "500-case run" is a canonical unit; the exercise defines the class-A target as `p95 ≤ 5 min for a 500-case run` and derives the per-class-A-run target proportionally for other sizes. A 5000-case run at 8 min is *inside* target; a 100-case run at 5 min is *out* of target.
- Class-C "per-row" is the shape that matches the online-loop pattern of one-row-per-trace scoring.

Error budget:

- Latency is a *distributional* budget. The SLO is a p95; the error budget is "hours per month the p95 blows target." Typical: 30 minutes per month per class.
- Burn: an hour of blown p95 → ticket; two hours → page.

Runbook: `runbooks/latency_blown.md`. First step: which class? Is the queue backed up (scheduler issue) or is the runner slow (adapter / vendor issue)? Remediation: shift rate-limit share across classes; escalate the vendor if latency is upstream; consider spilling to a secondary runner if the primary is degraded.

### SLI #3 — Cost-per-team drift

Measurement:

```
cost_wow_delta[cost_centre] =
    (sum(eval_runs.cost_usd WHERE cost_centre=X AND ended_at IN this_week)
   - sum(eval_runs.cost_usd WHERE cost_centre=X AND ended_at IN last_week))
   / sum(eval_runs.cost_usd WHERE cost_centre=X AND ended_at IN last_week)
```

Notes:

- Drift is signed. A -20 % drop is also a signal — usually a broken CI trigger or an online-loop that stopped sampling. Silent underspend is often the *scarier* failure because a working monitor going silent is invisible.
- Drift is normalised for known scale events. A team that opts into a `pricing_snapshot` change ("we moved from `claude-3-5-sonnet` to `claude-3-5-haiku` for online scoring") declares the scale-change in a config; the drift SLI ignores the week that includes the change.
- Excluded cost-centres are a bug. Every run must belong to a cost centre; a run that lands in the default cost centre (`unassigned/plane-default`) is an alert of its own.

Error budget:

- The cost-centre owner has an implicit budget of 20 % w-o-w tolerance. Above that, they get an automated email and a Slack ping.
- Above 50 % w-o-w drift on any cost centre, the eval team gets a ticket — this is the "the plane's cost accounting is broken" signal.

Runbook: `runbooks/cost_drift.md`. First step: is the drift real (team increased their evals) or is it a plane bug (pricing snapshot drift, missed reconciliation, un-attributed runs)? Remediation: for real drift, notify the owner and confirm the intent; for a plane bug, patch, re-cost, publish a correction.

### SLI #4 — Queue-drop rate

Measurement:

```
drop_rate_15m[class] =
    count(dropped_runs WHERE priority=class AND dropped_at > now() - 15m)
  / count(all_submissions WHERE priority=class AND submitted_at > now() - 15m)
```

Notes:

- A "dropped run" is one the plane refused to enqueue — capacity refusal, envelope refusal, quota hard-cap refusal, class-limit refusal. It is *not* a run that failed after starting (that is SLI #1).
- Class-A drops are *never* acceptable at any rate — a CI gate that cannot enqueue is a developer that cannot ship. Class-C drops at low rates are acceptable for cost-management reasons (the class-C bucket is intentionally scarcer).

Error budget:

- Class A: 0 drops. Any class-A drop pages.
- Class B: < 0.1 % over 15 minutes; sustained higher → ticket.
- Class C: < 0.1 % over 15 minutes; higher rates are typical during a quota transition and should be reviewed weekly.

Runbook: `runbooks/queue_drop.md`. First step: which class, what reason (capacity, envelope, quota)? Remediation: capacity → scale the runner workers; envelope → raise the vendor cap or shift share across classes; quota → this is not the eval team's problem, but the runbook links to the cost-centre owner.

### The platform's own dashboard

One dashboard, four tiles (one per SLI), three visible axes: current status (green / yellow / red), 7-day trend line, error budget remaining. Every tile has a link to the runbook.

Additional tiles that live below the SLIs:

- **Runner health.** One row per runner adapter: success rate, mean latency, cost per row. When one adapter degrades, the plane can shift traffic before the SLIs blow.
- **Cost-centre roll-up.** Top-20 cost centres by 7d spend; column for w-o-w delta with the SLI #3 threshold coloured.
- **Reproducibility check.** The chapter 02 SLI check running on a scheduled sample of historic runs. This is not one of the four core SLIs but it is *the* SLI that says the schema is doing what it promised. Green means the store is trustworthy; red means the last week's runs may not reproduce.
- **Pricing-snapshot age.** How old is the current `pricing_snapshot` for each vendor. A snapshot > 30 days old is a signal to reconcile against the vendor's current pricing page.
- **Test-case age.** For each suite, when was the last new case added, when was the last rot-check promotion. Suites with no additions in 90+ days get a slow-yellow warning (chapter 03's rot-check flow probably stopped).

The dashboard is the eval-team's own daily check-in. The mod-107 chapter 07 cross-persona dashboards look at the application; this dashboard looks at the platform that serves them.

### Error-budget policy

An SLO without a policy is a hope. The policy names what changes when the error budget is spent.

- **Budget healthy (> 25 % remaining).** Normal cadence. Adapter swaps, schema migrations, runner upgrades, new integrations proceed with normal review.
- **Budget low (5 – 25 % remaining).** Only necessary changes. Feature work pauses; the on-call resets budget through change-freeze reliability work.
- **Budget spent (≤ 0 %).** Change freeze on the plane, one week or until the SLI recovers, whichever is longer. The eval-team lead documents what caused the burn and what will prevent recurrence.

The policy is public. Product teams that read the SLO also read the policy — they know a change freeze is a real thing that happens, not "eval team's fault." This is what makes the SLIs credible.

### Capacity planning against the mod-107 sample-rate math

The plane's throughput is not a fixed number. It is a function of:

- The vendor's rate-limit envelope (chapter 05).
- The runner adapters' effective throughput (each has its own concurrency ceiling).
- The judge model's per-token latency.
- The class-scheduler weights.

The exercise ships a small capacity model — given the current envelope, the current adapter mix, and the current class weights, what is the plane's sustainable RPS per class? The mod-107 sample-rate arithmetic (chapter 02 of mod-107) reads this: the sample rate must not exceed the class-C sustainable RPS at 70 % utilisation (the 30 % headroom is what absorbs a class-A burst without starving class C).

Two rules the capacity model enforces:

- **The sum of class targets does not exceed the plane's total.** If class A is dimensioned for 5 rps of PR gates, class B for 2 rps of nightlies, class C for 8 rps of online loop — the plane must be able to sustain 15 rps at 70 % utilisation. If not, one class will starve under real load.
- **A new team joining the plane is a capacity decision, not a config change.** Onboarding a new cost centre with a projected 10 rps of class-C traffic requires the plane's capacity model to accommodate it. The eval-team lead reviews.

### Alerts, on-call, and the eval team's rotation

The plane's on-call is the eval team. The rotation:

- **Primary.** One eval-team engineer per week. Owns pages during business hours + first-response outside business hours. Runs the SLI review at the weekly ops meeting.
- **Secondary.** Backup for the primary. Escalation path when the primary cannot respond within a stated time (usually 15 minutes for a page).
- **Manager escalation.** Third layer for prolonged incidents (> 4 hours) or cross-team escalations (a runner vendor outage that needs a joint call).

Alerts route through the org's alerting surface (PagerDuty, Opsgenie, or the org's homegrown). Every alert has a runbook link in the alert body. The eval-team lead reviews the past week's alerts at the ops meeting; alerts that fire without follow-through are a signal (either the alert is noisy and gets tuned, or the runbook is inadequate and gets updated).

### Where the platform is *not yet* the org's primary eval-serving surface

Two clean scale-out limits, either of which is a delegation trigger (chapter 07):

- **Cross-modality demand.** A new product line wants image or audio evals. The slice's schema, adapter interface, and cost accounting extend without breaking; the runners the adapters wrap probably need to change (RAGAS is text-only; the DeepEval multimodal support is limited; a new runner may be required). This is a project scoped in months, and it is the peer's territory once the org has two or more modalities the slice needs to serve.
- **Cross-tenant isolation.** A new customer relationship requires per-tenant isolation of eval data, per-tenant quotas that come with contractual SLAs, and per-tenant billing. The slice's cost centre is a *soft* isolation; it is not a per-tenant hardened boundary. Turning the slice into a paying-customer service is the peer's territory.

Chapter 07 walks the escalation contract and the artefacts the slice hands over.

### The plane's own reliability practice: three rules

Three rules from the SRE tradition that the exercise makes explicit:

- **Every alert is actionable.** An alert that fires "for information" is not an alert; it is a metric. The runbook's first step is a discriminating question, not "check the dashboard." An alert that has fired three times without action gets tuned or removed.
- **Every incident produces a postmortem.** Not a document nobody reads — a specific artefact (`postmortems/YYYY-MM-DD-incident-slug.md`) with a timeline, a root cause, a set of action items each with an owner and a due date, and a follow-up review two weeks later.
- **Every SLO is reviewed quarterly.** Are the SLOs still the right ones? Are the error budgets tight enough (too tight = false-positive change freezes; too loose = the SLIs are theatre)? The SLO doc has a version and a `last_reviewed_at`.

## Summary

- The plane is a **production system**; four **platform SLIs** commit to it publicly and drive on-call.
- The four SLIs: **success rate ≥ 99.5 %** stratified by class, **latency p95 ≤ target** per class, **cost-per-team drift within ±20 % w-o-w**, **queue-drop rate ≤ 0.1 %** (0 for class A).
- Each SLI has an **error budget** with **fast / medium / slow burn-rate alerts** per the SRE workbook pattern.
- Error-budget policy: **healthy = normal cadence**, **low = only necessary changes**, **spent = change freeze** until the SLI recovers.
- The **platform dashboard** shows the four SLIs plus runner health, cost-centre roll-up, reproducibility check, pricing-snapshot age, test-case age.
- **Capacity planning** ties the plane's throughput to the mod-107 sample-rate math; new-team onboarding is a capacity decision, not a config change.
- The eval team's **on-call rotation** owns pages, weekly SLI reviews, and postmortems. Every alert has a runbook; every incident produces a postmortem; every SLO is reviewed quarterly.
- Two clean scale-out limits trigger delegation to `model-evaluation-engineer`: **cross-modality demand** and **cross-tenant isolation**. Chapter 07 walks the contract.

Chapter 07 draws the delegation boundary — what the slice hands over when the org outgrows a product-eval slice, what it does not hand over, and how to write the escalation.
