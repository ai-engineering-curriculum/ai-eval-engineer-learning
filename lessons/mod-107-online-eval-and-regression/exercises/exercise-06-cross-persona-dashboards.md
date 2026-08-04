# exercise-06: Cross-Persona Dashboards

**Estimated effort:** 1 hour

## Objective

Ship the three chapter 07 dashboards — **PM**, **on-call**, **safety-reviewer** — on top of the exercise-01 scored-row store, wire the alert channels appropriate to each persona, and walk each dashboard through with a proxy for the persona to confirm it can be read at a glance. Add the eval team's own loop-health dashboard.

This is a short exercise because the substrate is already in place. The point is to prove the three views are readable, not to build a new backend.

## Prerequisites

- Chapter 07 of this module.
- Exercise-01 scored-row store populated with ≥ 7 days of data across ≥ 2 cohorts and ≥ 2 rubrics.
- Exercise-02 drift alerts wired.
- Exercise-03 gate wired (optional; the on-call view degrades gracefully without it).
- A dashboarding surface: your trace-backend vendor's own dashboard (Phoenix, Langfuse, Weave, Braintrust) or a general-purpose tool your org already uses (Grafana, Superset, Metabase, Looker). If none is available, a static `mkdocs` or Streamlit page rendering the same charts is acceptable — the shapes are the deliverable, not the vendor.
- A Slack workspace (or Discord / Teams equivalent), a PagerDuty account or stub, and an email address for the PM digest.

## Set-up

1. Create `eval/online/dashboards/` with a subdirectory per persona:

   ```
   eval/online/dashboards/
   ├── pm/
   │   ├── README.md
   │   └── dashboard.json           # vendor export or Grafana JSON
   ├── on_call/
   │   ├── README.md
   │   └── dashboard.json
   ├── safety_reviewer/
   │   ├── README.md
   │   └── dashboard.json
   ├── loop_health/
   │   ├── README.md
   │   └── dashboard.json
   └── channels.yaml
   ```

2. `channels.yaml` declares the per-persona alert routing:

   ```yaml
   pm:
     email_digest: pm-list@company.example
     schedule: weekly-monday-0900
     slack_channel: '#support-bot-eval-weekly'
   on_call:
     pagerduty_service: support-bot-eval-oncall
     slack_channel: '#support-bot-eval-alerts'
     severity_page: block, block-and-page, block-and-auto-rollback
     severity_slack: warn
   safety_reviewer:
     review_meeting: weekly-thursday-1400
     escalation_channel: '#ai-safety-escalations'
     severity_page: never   # policy work, not incident work
   eval_team:
     slack_channel: '#eval-team-loop-health'
     schedule: continuous
   ```

## Requirements

Produce a PR against your working branch that adds:

1. **PM dashboard** in `pm/`:
   - Headline health tile per rubric (7-day weighted mean vs same window a month ago; green / yellow / red against pre-registered thresholds).
   - 90-day primary-metric trend with the floor drawn as a horizontal line.
   - Small-multiples of the primary metric per top-5 cohort by traffic.
   - Experiment ledger (exercise-05 pre-registrations, primary readout, decision).
   - Change ledger (merges to `/eval/`, `/prompts/`, `/chains/` with primary-metric delta co-located).
   - Refresh on load only; no auto-refresh; no CI clutter.
   - `pm/README.md` documents the layout, the audience, and the digest schedule.
2. **On-call dashboard** in `on_call/`:
   - Active alerts (metric, cohort, severity, since, runbook link).
   - Rollback bank inventory (feature flag names + state, deploy tag, retrieval index hash, model snapshot).
   - Live 60-minute metric strip for the five ship-critical metrics per in-flight cohort; auto-refresh every 30 seconds.
   - Related-incidents list (last 30 days, same metric or cohort).
   - One-click link to the mod-102 trace explorer for the current failing cohort.
   - `on_call/README.md` documents the layout, the alert-routing wiring, and the "read in 30 seconds" contract.
3. **Safety-reviewer dashboard** in `safety_reviewer/`:
   - Cohort × safety-metric matrix (current 30-day mean, colour vs the safety floor).
   - Suppression register (currently-suppressed safety alerts with rationale + follow-up).
   - Cohort coverage report (effective sample size per cohort per safety metric; below-floor rows highlighted).
   - Injection / adversarial-input rate 90-day trend per surface (upstream classifier signal).
   - Cross-reference to mod-109 review queue (count, median time-in-queue, oldest unresolved).
   - `safety_reviewer/README.md` documents the layout and the review-meeting cadence.
4. **Loop-health dashboard** in `loop_health/`:
   - Actual sample rate per surface vs. target.
   - Judge-tier composition (`f_OSS / f_mid / f_frontier`) vs. plan.
   - Monthly spend to date vs. cap.
   - `judge_missed` rate per surface.
   - Aggregator lag per window size.
   - Alerts per severity per week.
   - Median time-to-runbook-resolution per severity.
   - `loop_health/README.md` names the eval team as the primary reader.
5. **`channels.yaml`** — the routing declaration above, and the wiring in code:
   - PM digest sender (a cron job that emails the PM dashboard link + a summary tile per Monday morning).
   - On-call PagerDuty routing for severity=block / severity=block-and-page / severity=block-and-auto-rollback.
   - On-call Slack routing for severity=warn.
   - Safety-reviewer weekly meeting agenda-item template that pulls the current cohort × safety-metric matrix.
6. **Persona walk-through evidence.** For each dashboard, hand the URL to a proxy (a teammate, or yourself with fresh eyes 24 hours later) and time the following:
   - PM: reads the headline health tiles and identifies the primary-metric trend direction in ≤ 60 seconds.
   - On-call: identifies the top active alert, opens its runbook, and identifies the correct rollback knob in ≤ 30 seconds.
   - Safety-reviewer: identifies any red cell in the cohort × safety-metric matrix and cross-references it against the suppression register in ≤ 90 seconds.

   Record the timings in each `README.md`; if the timings exceed the targets, iterate on the layout.

## Starter guidance

- **Do not build a master dashboard.** The whole exercise is the *split*. Any temptation to consolidate is the anti-pattern; resist it.
- **Refresh cadence is the persona.** Weekly digest for the PM. 30-second refresh for the on-call. On-demand for the safety reviewer. Match the cadence to the persona; the wrong cadence is worse than the wrong chart.
- **Runbook link on every alert row is mandatory.** An on-call alert without a runbook link is an alert that ends in a hallway conversation. This is chapter 07's non-negotiable.
- **Safety metrics are cohort-first.** Aggregate is *additional*. If your matrix defaults to aggregate rows, you are already off the target.
- **PagerDuty for block; Slack for warn.** Do not degrade PagerDuty by routing warns to it. Do not amplify Slack by routing blocks to it exclusively. The channel is part of the severity semantics.
- **Vendor-native first; Grafana second.** Your trace backend vendor already has a dashboard surface tailored to its data model. Use it when the persona's view maps cleanly; wrap it with Grafana / Superset / Metabase when the persona's view needs primitives the vendor does not ship.
- **The eval-team loop-health dashboard is not decorative.** When the loop is degraded, every downstream persona sees under-sampled numbers and trust erodes. Rotate this dashboard through the eval team's stand-up.

## Acceptance criteria

You are done when:

- All four dashboards are live on the chosen surface and reachable at durable URLs.
- Alert channels are wired per `channels.yaml`; a synthesised severity=block alert reaches PagerDuty; a synthesised severity=warn alert reaches Slack; the PM digest email delivers on the declared schedule.
- The three persona walk-through timings are recorded and meet the targets (PM ≤ 60s, on-call ≤ 30s, safety reviewer ≤ 90s). Iterate on any that miss.
- Access-control posture is documented: PM sees aggregates only; on-call and safety reviewer see raw traces under audit log; PII scrubbing (mod-102 chapter 06) is the layer everyone reads through.
- The eval team loop-health dashboard is on the eval team's stand-up rotation.

## Stretch goals

- **Cross-vendor dashboard portability.** Export the same three dashboards in two vendors (e.g., Phoenix and Grafana); confirm the on-call reads them the same way; document which primitives are vendor-portable and which require re-authoring.
- **Digest analytics.** Instrument the PM digest — was it opened? was the link clicked? how many times per week? Feed this into a monthly report on dashboard usage; retire dashboards nobody reads.
- **Runbook completeness gate.** Add a CI check that every metric named in `thresholds.yaml` has a corresponding runbook file; fail CI on a missing pairing. Prevents the "we shipped a metric with no runbook" drift.
- **Silent-cohort report.** Add a scheduled report to the safety-reviewer view listing cohorts whose sample size fell below the coverage floor in the last 30 days; a cohort the loop is not observing is a cohort the loop cannot alert on.
- **Mobile-optimised on-call view.** The on-call is often reading at 3 a.m. on a phone. A stretch goal: a mobile-first variant of the on-call view; test it on an actual phone screen.
- **Cross-link with mod-109 review queue.** When a safety-reviewer clicks a red cell in the matrix, one link opens the mod-109 review-queue filter for that cohort × metric. Two clicks total from the matrix cell to a review item.
- **Digest-to-decision loop.** For each PM digest, capture a 1-sentence "what changed and what did we do about it" note in the following week's digest. Turns the digest from a read-only artefact into a decision-loop artefact.

## What this exercise does *not* cover

You are not building the safety rubrics (mod-108) or the human-review queue (mod-109); the safety-reviewer dashboard reads them but does not implement them. You are not building the mod-110 eval-data-platform underneath; the dashboards read the exercise-01 store as substrate. This exercise closes the module by ensuring the humans downstream of the loop can act on it.
