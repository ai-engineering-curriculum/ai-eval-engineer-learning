# Cross-Persona Dashboards: PM, On-Call, and Safety Reviewer Views

## Motivation

Chapters 02 – 06 built a rich scored-row store, a drift monitor, a cohort gate, an anytime-valid inference layer, and an A/B interface. Every component emits data. Zero of that data is useful if the humans who need it cannot read it.

Three humans read the online-loop output regularly, and their needs almost never overlap.

- The **product manager** wants to know: *is the surface healthy? which cohort is best? did our recent experiment ship? what changed this week?* Time horizon: weeks. Sample size: aggregate. Cadence: reads on Mondays; not paged.
- The **on-call engineer** wants to know: *what is currently on fire? what is the runbook? which rollback flag do I flip?* Time horizon: the current shift. Sample size: whatever is significant *now*. Cadence: paged.
- The **safety reviewer** wants to know: *did any safety metric regress on any cohort in any recent window? are there any suppressed alerts I should un-suppress? which product surface has the highest injection-attempt rate this quarter?* Time horizon: a review cycle (weekly or monthly). Sample size: aggregate. Cadence: reviews on a schedule; escalates if something crosses a policy threshold.

One dashboard cannot serve all three. Attempting to build a "master dashboard" produces a page that overwhelms the PM, buries the on-call's rollback flag under weekly trend charts, and hides the safety reviewer's cohort-slicer behind an aggregate that averages the safety signal into noise.

This chapter walks the three per-persona dashboards, states what each needs to answer, calls out the anti-patterns each falls into, and maps the design to the vendor UIs (Arize Phoenix, Langfuse, W&B Weave, Braintrust) that most teams build on top of.

## Core concepts

### The PM dashboard: aggregate health and week-over-week

Purpose: answer "is the surface healthy?" without leaving the page. Read on a laptop, not a phone. Loaded once a week; the PM does not tab through six views.

Layout (top to bottom):

1. **Headline health tile.** Per rubric metric, current 7-day weighted mean vs. the same 7-day window a month ago. Green / yellow / red per the pre-registered thresholds file. Zero baby-picture charts; one number per metric.
2. **Trend chart, primary metric.** 90 days of the primary metric with a 7-day rolling mean and the pre-registered floor drawn as a horizontal line. One chart, not per-cohort.
3. **Cohort breakdown.** A small-multiples grid of the primary metric per top-N cohort (top 5 by traffic). Same 90-day trend per panel. This is where a locale regression becomes visible in the PM's peripheral vision.
4. **Experiment ledger.** A table of experiments the surface ran in the last quarter, primary metric readout, ship / iterate / kill decision, link to the pre-registration (chapter 06). Rows sorted by decision date.
5. **Change ledger.** A table of merges to the eval-relevant paths (prompt / chain / retriever / model-snapshot bump) in the last quarter, with the primary-metric delta co-located. Correlation-not-causation caveat in the header.

Anti-patterns for the PM view:

- **Per-minute or per-hour granularity.** A PM does not act on a 15-minute wobble; the dashboard should not show one. Downsample to daily.
- **Confidence intervals on every number.** The PM does not read CIs; the eval team does. Show point estimates in the PM view; expose CIs on hover or a drill-down.
- **Auto-refresh.** A dashboard that refreshes while the PM is reading it is a dashboard the PM stops trusting. The PM view refreshes on load only.

### The on-call dashboard: what is on fire, right now

Purpose: answer "what is broken and what do I do about it?" in under 30 seconds. Read on a phone at 3 a.m. as often as on a laptop.

Layout (top to bottom):

1. **Active alerts.** One row per firing alert: metric, cohort, severity, since, runbook link. Link is one tap away; runbook has the rollback command as step 1 (chapter 04's rollback shape naming).
2. **Rollback bank.** A per-surface list of the active rollback knobs: feature flag names + current state (on / off), deploy tag currently serving, retrieval index hash currently active, model snapshot currently pinned. This is the on-call's inventory of "what can I flip."
3. **Live metric strip.** The five ship-critical metrics (primary quality, primary safety, cost p95, latency p95, refusal rate) for the last 60 minutes, per cohort that is currently in flight. Auto-refresh every 30 seconds — this view *is* about the current moment.
4. **Related incidents.** Recent (last 30 days) alerts on the same metric or cohort, with resolution notes. Prevents re-solving a solved problem.
5. **The escape hatch.** A one-click link to the mod-102 trace-explorer for the current cohort, pre-filtered to failing traces. When the runbook does not resolve, the on-call needs the raw traces.

Anti-patterns for the on-call view:

- **Weekly trends.** A weekly chart is what the PM is looking at; it is not what an on-call at 3 a.m. is looking at.
- **Alerts without runbook links.** An alert that does not link a runbook is an alert that becomes a hallway conversation. Runbook link is mandatory.
- **Too many active-alert rows.** More than 5 – 10 rows means the alerting policy is over-firing; fix the policy, do not add a scrollbar.

### The safety reviewer dashboard: policy compliance and cohort coverage

Purpose: answer "did any safety metric regress anywhere?" and "am I covering the cohorts I claim to cover?" on a review cycle.

Layout (top to bottom):

1. **Safety-metric matrix.** Rows = safety rubrics (jailbreak resistance, PII-leakage rate, refusal correctness, harmful-content rate — mod-108's rubric list). Columns = pre-declared cohorts (locale × tenant tier × product surface). Cells = current 30-day mean; colour coded against the safety floor. Any red cell must be un-hidable, un-suppressible without a rationale.
2. **Suppression register.** Every currently-suppressed safety alert: metric, cohort, suppressed by, suppressed until, rationale, follow-up issue. Reviewer's job is to challenge each entry.
3. **Cohort coverage report.** For every pre-declared cohort, the effective sample size on each safety metric in the last 30 days. Below-floor coverage rows are highlighted — the reviewer needs to know when a cohort is invisible to the loop, not just when a cohort has a bad number.
4. **Injection / adversarial-input rate trend.** Rate of prompts flagged by the upstream input classifier (mod-108's injection detector), 90-day trend per surface. This is a leading indicator; a spike here often precedes a safety metric regression by days.
5. **Cross-reference to review queue.** Count of items currently in the mod-109 human-review queue tagged `safety`; median time-in-queue; oldest unresolved item. The reviewer's follow-through on this queue is a program metric.

Anti-patterns for the safety-reviewer view:

- **Aggregation that hides a cohort.** Safety metrics are cohort-first. Aggregate rows are additional, not primary.
- **Point-in-time snapshots without a diff.** Compare the current 30-day mean to the previous 30-day mean, per cell. A number without a compare is not a review artefact.
- **A "safety score" that averages rubrics.** No single scalar covers jailbreak + PII + refusal + harmful-content; averaging them together creates a number that hides its worst constituent.

### Vendor-specific mapping

The three views map onto the current dashboard surfaces of the trace-backend vendors. Pick the vendor whose default dashboard shape most closely matches the persona you serve; wrap the others with the same underlying store (chapter 02's scored-row schema is vendor-neutral).

- **Arize Phoenix.** Trace-focused; strong on the on-call view (per-trace drill-down, run comparisons, span-level filtering). Dashboards page supports the aggregate views for PM and safety-reviewer; embedding scores requires the OTel-annotations pattern from chapter 02.
- **Langfuse.** Score-first; strong on the PM view (native metric aggregation, user-defined dashboards, cohort filters). On-call view via the "traces" view plus a Slack integration for alerts. Safety-reviewer via saved views on tagged scores.
- **W&B Weave.** Experiment-first; strong on the PM experiment-ledger and the eval-team's calibration workflow. On-call and safety-reviewer views require more custom dashboard authoring; the strength is in the `weave.Evaluation` primitive rather than the dashboard surface.
- **Braintrust.** Experiment-and-comparison-first; strong on the PM experiment ledger and the "compare two runs" view. Online monitoring dashboards are the newest surface; the on-call view is easier once online monitoring is fully wired up.

If you have a general-purpose dashboarding tool (Grafana, Superset, Metabase, Looker) already used by the org, the correct default is often to publish the scored-row store to that tool and build the three views there — the org has muscle memory around the tool, and no vendor UI matches a well-authored Grafana board for the on-call use case.

### Alert channels: match the channel to the persona

Persona × channel mapping is a small design that pays dividends.

- **PM.** Weekly email digest of the PM dashboard. No pages. Slack channel `#surface-name-eval-weekly` receives the digest link.
- **On-call.** PagerDuty (or Opsgenie, Splunk On-Call) for severity=block. Slack channel `#surface-name-eval-alerts` for severity=warn. Runbook link embedded in the payload.
- **Safety reviewer.** Weekly review meeting with the safety-reviewer dashboard as the standing agenda item. Ad-hoc escalation via a named channel (`#ai-safety-escalations`), *not* PagerDuty (safety escalations are policy work, not incident work).

Anti-patterns to avoid:

- **All personas subscribed to all channels.** Every persona learns to filter out most channels and eventually all of them.
- **Every alert goes to Slack.** Slack degrades to background noise; PagerDuty is what commands attention. Reserve PagerDuty for severity=block; use Slack for severity=warn.
- **No digest for the PM.** A dashboard the PM must remember to open is a dashboard the PM opens once a quarter. The digest is what brings the PM to the dashboard.

### The eval program's own dashboard

One dashboard the eval team reads, not the personas above: **loop health**. This is where you see whether the loop itself is functioning.

- Sampled-rate per surface vs. target (chapter 02's `p`).
- Judge-tier composition (`f_OSS / f_mid / f_frontier`) vs. plan.
- Monthly spend to date vs. cap.
- `judge_missed` rate per surface.
- Aggregator lag per window size.
- Rate of alerts per severity per week.
- Median time-to-runbook-resolution per severity.

The eval team's job includes maintaining this dashboard's health; when the loop is degraded, the personas' dashboards are showing under-sampled numbers and none of the downstream trust holds. Rotate this dashboard through the eval team's stand-up.

### Access control and PII posture

Chapters 03 and 04 assumed the cohort keys and trace content are already in the trace backend. Two access-control notes.

- **Trace content in dashboards.** A PM does not need per-trace content in a dashboard. Give the PM aggregate numbers and cohort breakdowns; keep raw prompts and responses behind a role that requires justification. This is a PII-defence posture and an SOC 2 / ISO 27001 posture.
- **Safety-reviewer access to raw failing traces.** The safety reviewer *does* need per-trace content for red-flagged samples. Their role grants that access with an audit log. Mod-102 chapter 06 defined the PII-scrubbing policy; the safety reviewer reads through the scrubbed layer, not the raw.
- **On-call access.** On-call needs raw content for the current-incident cohort; the audit log captures who queried what for the incident retrospective. This is standard incident-response posture.

## Summary

- Three humans, three dashboards, one underlying store. Do not build a master dashboard.
- **PM view**: aggregate health tiles, one primary-metric trend, cohort small-multiples, experiment ledger, change ledger. No per-minute granularity; no auto-refresh; no CI clutter.
- **On-call view**: active alerts with runbook links, rollback bank inventory, live 60-minute metric strip, related-incidents list, one-click trace-explorer escape. Auto-refresh; runbook-link mandatory.
- **Safety-reviewer view**: cohort × metric matrix, suppression register, cohort-coverage report, adversarial-input-rate trend, cross-reference to mod-109 review queue. No aggregate-only rows on safety.
- **Vendor mapping**: Phoenix (on-call strengths), Langfuse (PM strengths), Weave (experiment-first), Braintrust (comparison-first). A general-purpose Grafana / Superset board is often the right default for the on-call view.
- **Alert channels** match the persona: email digest for PM, PagerDuty for on-call severity=block, weekly review meeting for the safety reviewer.
- The eval team runs its own **loop-health** dashboard: sample rate, tier composition, spend, `judge_missed`, aggregator lag, alerts-per-week, time-to-resolution.
- **Access-control posture**: PM sees aggregates; on-call and safety-reviewer see raw traces under audit; PII scrubbing (mod-102 ch. 06) is the layer everyone above the raw store reads through.

The module ends here. Exercises 01 – 06 build the artefacts — sampler, drift alerts, canary gate script, confidence-sequence library, A/B pre-registration, and the three dashboards — that mod-108 (safety-specific rules), mod-109 (human review), mod-110 (eval data platform), mod-111 (cost / latency / quality), and mod-112 (program posture) will consume as first-class inputs.
