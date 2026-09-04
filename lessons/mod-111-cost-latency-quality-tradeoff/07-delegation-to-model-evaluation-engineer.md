# Delegation to Model-Evaluation-Engineer and AI-Infra-Performance: Where App-Altitude Ends

## Motivation

Six chapters ago, the module declared its scope: **the app-altitude trade-off surface**. Not model-altitude benchmarks (MMLU, GPQA, HumanEval, MMMU). Not fleet-altitude kernels (MLPerf-shaped serving, GPU-hours-per-token). The AI Evaluation Engineer owns quality-as-users-experience-it, cost-as-the-invoice-lines-say, latency-as-the-app-p95-shows.

That boundary is easy to hold on day one and easy to lose on day 200 — the report works, product signs, someone asks "so should we swap to Llama-4 when it lands?" and the eval team, wanting to be helpful, drifts into a model-altitude answer they are not staffed for. Six months in, the eval team is running MMLU sweeps on every vendor snapshot, arguing with the fleet team about batching config, and neither the app-altitude trade-off report nor the peer's actual work is getting done well.

This chapter is the anchor. It walks:

- Where the app-altitude surface ends and each peer's remit begins.
- The escalation shape — when to invite `model-evaluation-engineer` or `ai-infra-performance` in, what to hand them, what *not* to.
- The reconciliation contract — how the peer's numbers flow into the app-altitude report and how the app-altitude gaps flow back to the peer.
- Failure modes on both sides of the boundaries.

The chapter is short by intent. Its whole job is to make three boundaries crisp before they blur under pressure.

## Core concepts

### The two peers at a glance

Two distinct peer tracks; two distinct remits; two distinct escalation shapes.

| Axis | This module (app altitude) | `model-evaluation-engineer` (model altitude) | `ai-infra-performance` (fleet altitude) |
|---|---|---|---|
| **What it measures** | Quality on cohorts, cost per feature, TTFT / TPOT / streaming p95 as users see them, safety in prod | Offline benchmark suites (MMLU, GPQA, HumanEval, MMMU, LiveCodeBench), MLPerf Inference (Server / Offline), model capability profiles | Kernel throughput, batching efficiency, KV-cache eviction, GPU-hours per token, node utilisation, capacity plan |
| **What it does not** | Model-altitude offline benchmarks; fleet-altitude kernels | App-altitude cohort preservation, per-feature cost, app-layer retries | App-altitude quality, per-feature cost attribution, cohort analytics |
| **Owner role** | AI Evaluation Engineer (this track) | Model Evaluation Engineer (Model-Development family, L30) | AI Infra Performance Engineer |
| **Primary artefacts** | Trade-off report, routing report, replacement-regression report, app-altitude latency report, per-feature cost report | Benchmark scorecards; capability profile per model family; MLPerf Inference submission; distillation quality analysis (upstream side) | Serving benchmarks per model / hardware combo; kernel selection; batching config; capacity plan; SLOs on the serving fleet |
| **Time horizon of a release** | Weeks per candidate; days per swap-in | Weeks per benchmark suite refresh; months per model-family capability profile | Weeks per kernel or batching change; quarters per capacity plan |
| **Escalation cadence** | Per shipping decision (weekly rhythm) | Per model-family release (monthly – quarterly rhythm) | Per capacity or performance regression (monthly rhythm) |

The peers are not "one level up" or "the deeper version" of the same job. They are different jobs. The transition from app-altitude gap to peer work is a **hand-off** at a specific altitude, not a promotion.

### When to escalate to `model-evaluation-engineer`

Three triggers.

**Trigger 1 — a candidate model needs a capability profile the org does not have.** The trade-off report is being built for a swap-in from GPT-family to Claude-family (or vice versa, or to an open-source model). Before the app-altitude eval, someone needs to know: is this model plausibly good enough at code / math / reasoning / long-context / tool-use for this feature? That is a model-altitude question. The peer runs MMLU / GPQA / HumanEval / MMMU / LiveCodeBench / whatever benchmark tests the capability the feature depends on. The app-altitude eval reads the peer's capability profile and *then* runs the paired-comparison eval.

If the peer has not profiled the candidate model, that is a blocker on the app-altitude eval, not something the eval team should paper over by inventing a mini-benchmark. Escalate.

**Trigger 2 — a distillation project needs an upstream teacher-choice or student-training decision.** Chapter 04 covered distillation as one shape of replacement. The *training* of the student is outside this module's scope — the peer or the model-training team owns it. The app-altitude eval reads the student and produces the coverage-and-quality plot; it does not choose which teacher, which training data, or which distillation objective. When the app-altitude report says "the student under-performs on cohort X," the escalation is to the peer or the training team, not a fix the eval team implements.

**Trigger 3 — an app-altitude regression is unattributable to the app layer.** The trade-off report shows a quality regression on a cohort. The eval team checks the prompt (unchanged), the chain (unchanged), the retriever (unchanged), the decoding config (unchanged). The `model_snapshot` bumped a week ago. The regression is model-altitude — the vendor released a snapshot that regressed on this cohort. Escalate: the peer's capability profile refresh is the fix.

Anti-trigger: **"we should run our own MMLU."** No. MMLU is a public benchmark; the peer subscribes to LMSys leaderboards / HuggingFace evaluations / vendor-published numbers. If the peer does not exist yet in the org, the trade-off report cites public sources with the caveat "as reported by the vendor" — it does not invent its own model-altitude benchmark to substitute for the peer.

### When to escalate to `ai-infra-performance`

Three triggers.

**Trigger 1 — the app-altitude latency reconciliation attributes a sustained delta to fleet queue or serving throughput.** Chapter 05's reconciliation footer names the delta attribution: vendor p95 320 ms, app-altitude p95 480 ms, +45 ms attributable to fleet queue. If the fleet-queue attribution grows over time (60 → 90 → 130 ms), the fleet is under-provisioned for the load. Escalate to the fleet peer with the app-altitude data as evidence. Do not build your own scheduler.

**Trigger 2 — a self-hosted model's serving cost is dominated by an under-utilised fleet.** The per-feature cost report (chapter 06) shows an OSS or in-house model whose per-token cost is higher than the vendor's for a comparable model. The fix is fleet-level (batching, KV-cache, kernel choice, autoscaler tuning) — not app-level. Escalate.

**Trigger 3 — an MLPerf-shaped benchmark shows the model is fine but the fleet's throughput is far below the benchmark.** The peer produces MLPerf Inference numbers on the org's hardware. The eval team's app-altitude p95 is 3× the peer's benchmark. The gap is fleet-attributable. Escalate with the trace-derived latency breakdown and the peer's benchmark as evidence.

Anti-trigger: **"we should tune batching ourselves."** No. Batching, KV-cache eviction, kernel selection are fleet-owned. If the fleet peer does not exist in the org yet, the eval team's report names the fleet-attributable delta and hands it to whichever infra function is closest — but does not fix it directly.

### The escalation ticket

When one of the triggers fires, the escalation is a specific ticket. Two shapes — one per peer.

**Model-altitude escalation to `model-evaluation-engineer`:**

```markdown
# App-altitude regression / gap — model-altitude escalation

## Triggering signal
- Trade-off report: <link to the report>
- Cohort with regression: <cohort, current quality, prior quality, magnitude>
- Model snapshot at time of regression: <model_snapshot>
- App-layer investigation ruled out: prompt (unchanged as of <hash>),
  chain (unchanged as of <hash>), retriever index (unchanged as of <hash>),
  decoding config (unchanged as of <hash>).

## Ask
- Capability profile refresh on <model_snapshot> vs. prior snapshot on
  benchmarks relevant to <cohort>'s task shape (specifically: <benchmark_names>).
- If a regression on the benchmark reproduces the app-altitude regression,
  that is the fix path.

## What this module hands over
- The paired eval set (mod-106 replay bundle) — cases where the regression
  reproduces at case level.
- The per-cohort quality delta with per-case detail.
- The lineage-key set (mod-110 chapter 02) that pins the exact configuration.

## What this module does not need from the peer
- App-altitude latency changes (owned here).
- Per-feature cost recalculation (owned here).
- Rubric authorship (owned by the product team; mod-104).

## Timeline
- Target: capability profile refresh within <business-days>.
- If the peer's refresh confirms model-altitude regression: escalate to vendor
  or trigger the routing eval (chapter 03) to route affected cohort back to
  prior snapshot.
```

**Fleet-altitude escalation to `ai-infra-performance`:**

```markdown
# App-altitude latency regression — fleet-altitude escalation

## Triggering signal
- App-altitude latency report: <link>
- Cohort with regression: <cohort, current p95, prior p95, magnitude>
- Reconciliation footer attribution: fleet-queue delta grew from
  <X ms> to <Y ms> over <window>.

## Evidence
- Trace samples (10 – 50) from cases in the regressed p95 bucket, with the
  fleet-queue span attribute populated.
- The peer's most recent MLPerf-shaped serving benchmark, for reference.
- The autoscaler / capacity dashboard for the affected serving pool.

## Ask
- Root-cause on the queue-time growth: is it a batching config, a
  KV-cache eviction pattern, a scale-out lag, a hardware degradation?
- Capacity plan revision if the growth is load-driven and sustained.

## What this module hands over
- The app-altitude latency report with per-cohort p95 shift.
- The mod-102 trace samples with full span attributes.

## What this module does not need from the peer
- App-altitude cohort quality changes (owned here).
- Per-feature cost changes downstream of the latency (owned here).
- Product decisions on rollback (owned here).

## Timeline
- Target: attribution within <business-days>; fix plan within a further <business-days>.
- If unresolved beyond the window: the trade-off report's rollback trigger fires
  and the affected cohort routes to a lower-latency alternative candidate.
```

Both tickets are formal artefacts — they land in the peer's tracker, they are reviewed at a joint meeting, they produce a decision. A slack DM that says "hey, is the fleet slow?" is not an escalation.

### The reconciliation contract — bidirectional

The delegation is not one-way. Numbers flow in both directions.

**Peer → app-altitude report.** The report reads down from the peers:

- The model-altitude capability profile — as a footnote on the quality column ("baseline capability: MMLU 78.4 %, HumanEval pass@1 62 % — sourced from `model-evaluation-engineer` capability sheet v14").
- The MLPerf-shaped serving benchmark — as the reconciliation footer on the latency column ("MLPerf Inference p99 first-token 380 ms on our hardware; app-altitude p95 480 ms; +100 ms attributed to fleet queue and app-layer overhead").
- The capacity plan — as the constraint set on the routing eval ("routing tier X's capacity is Y RPS at 95 % headroom; the routing report's fallback rate honours the constraint").

**App-altitude report → peer.** The report publishes back up:

- Cohort-specific regressions the model-altitude benchmark did not surface — the peer's next capability profile adds a cohort-relevant benchmark.
- Latency delta attributable to fleet queue — the fleet peer's next capacity revision uses it as input.
- Per-feature cost lines the fleet peer can act on — a feature whose per-token cost is out of line with the fleet benchmark is a candidate for fleet-side kernel optimisation.

Both directions of flow are contracts. The peer expects the app-altitude report to publish upward on a regular cadence (monthly is common); the app-altitude report expects the peer's capability profile and serving benchmark to be refreshed on a matching cadence. Chapter 07 of mod-110 covered the "the escalation ticket names the artefacts you hand over"; the same discipline here means the peer's artefacts show up in the app-altitude report by reference, not by re-derivation.

### "We outgrew our scope" vs "we mis-sized our scope"

Two very different situations that both feel like "we need to own more."

**Outgrew.** The org now runs enough LLM apps that the app-altitude trade-off surface generates real value; the peer functions exist and are staffed; the reconciliation contracts are working. The eval team's remit is stable and healthy. This is not an outgrow; it is a mature state.

**Mis-sized.** The peer functions do not exist in the org yet. The AI Evaluation Engineer is de facto the only person doing model-altitude or fleet-altitude work because no one else is doing it. This is a *staffing* problem the eval team names (and the org fixes by hiring the peers), not a scope problem the eval team solves by absorbing the work permanently.

Discriminating signals:

- If the fix is "hire a model-evaluation-engineer" or "hire an infra-performance engineer" or "stand up the peer function" — mis-sized. Name it; do not absorb.
- If the fix is "the peer function exists but is not producing the artefact I need" — that is a *reconciliation* problem, not an absorption problem. Escalate for the artefact; do not build it in the eval team.
- If in doubt, do the smaller ask first (the peer produces the specific artefact you need for the specific decision at hand) and reassess in 60 days. Absorbing peer scope is expensive on both sides.

The eval team should be willing to *bootstrap* peer artefacts when the peer function is being stood up — e.g., produce a first-cut capability profile for a candidate model so the shipping decision can proceed, with an explicit hand-off timeline. Bootstrap ≠ own; the hand-off is written into the ticket.

### The peers' stance

The peers are not sitting waiting to absorb escalations. Two things this changes:

- **Escalate early, in prose.** A ticket that surprises the peer with "please refresh the MMLU sweep on 12 vendor snapshots by Friday" is a ticket that gets declined. A conversation two weeks ahead that asks "we are going to need capability profiles on snapshots A, B, C by month-end for a Q4 shipping decision — what would you need from us to prioritise?" is what works.
- **Be honest about what you have already tried.** A ticket that describes an app-altitude regression without noting that the prompt just changed is a ticket that wastes the peer's cycles reproducing the regression. The eval team's investigation trail is part of the escalation.

### Where the eval team continues to matter after an escalation

Even after the peer accepts:

- **The trade-off report is still the AI Evaluation Engineer's.** The peer produces inputs; the report is authored here and signed by product here.
- **The cohort roster is still the eval team's.** The peer's capability profile does not choose cohorts; product surfaces do. The eval team maintains the cohort list.
- **The per-feature cost report is still the eval team's.** The fleet peer's benchmarks feed one row; the report is authored here.
- **The pre-registered decision rule is still the eval team's and product's.** The peer does not choose ship / hold / reject; product signs.

The escalation transfers ownership of *the specific altitude artefact*, not of *the trade-off decision*. Only the first is delegated.

### Failure modes to avoid

**On the eval team's side.**

- **Scope creep upward.** The eval team starts running MMLU / MLPerf sweeps because "we might as well have the number." Six months in, the eval team is doing the peer's job with the eval team's headcount. Correct: stop, escalate.
- **Scope creep sideways into product.** The eval team starts writing product prompts. Product ownership boundaries are separate; mod-104's rubric authorship discipline extends here.
- **Under-scoping.** The report ships without the reconciliation footer, or without the cohort matrix, or without per-feature attribution. The report is un-defendable. Correct: prioritise the chapter-02 to chapter-06 discipline over new surfaces.

**On the peer's side.**

- **Peer refuses to reconcile.** The peer's benchmark says the model is fine; the app-altitude report says it is regressing on a cohort; the peer's response is "the model is fine." The correct response is "the model is fine on the benchmark and regresses on your cohort — here is what your cohort exposes that the benchmark does not." Reconciliation is a dialogue, not a verdict.
- **Peer wants to own the trade-off report.** The peer's benchmark suite is deep; the peer wants to author the shipping decision. Correct: the trade-off report is app-altitude; product signs; the peer's benchmark is an input, not the decision.
- **Peer absorbs the cohort roster.** The peer's capability profile ends up including cohorts the eval team pre-declared. The cohort roster is the eval team's; the peer's benchmarks are chosen to match, not to replace.

### The delegation-boundary check the module runs quarterly

A quick, mechanised check the eval-team lead runs at the quarterly review.

- Have any of the six triggers (three per peer) fired since the last review? For each, is there an open escalation? If not, why not?
- Has the eval team taken on any work that plausibly belongs to a peer? (Ran an MMLU sweep, tuned a batching config.) If yes, was the addition reviewed against this chapter's guidance? If not, hand off or explicitly bootstrap with a timeline.
- Are the reconciliation contracts current? (Capability profiles refreshed within the SLA; MLPerf serving benchmarks refreshed within the SLA; per-feature cost report cited in the peer's inputs.) If sustained lag, escalate for the artefact.
- Are the trade-off reports being signed by product on the cadence the shipping decisions require? (The eval team's job is producing signable reports on time. If sign-offs slip, the report shape or the process needs adjustment.)

The check is a rubric. Running it quarterly is what keeps the three boundaries explicit.

### The end-of-module posture

By the end of this module you own the app-altitude trade-off surface. Concretely:

- You produce trade-off reports for the shipping decisions your product surfaces make — quarterly major decisions, weekly minor ones.
- You escalate cleanly to the model-altitude peer for capability profile refreshes and model-attributed regressions.
- You escalate cleanly to the fleet-altitude peer for latency-attributable-to-fleet regressions and per-feature-cost issues rooted in serving.
- You maintain the reconciliation contract — peer numbers flow into your report; your report's cohort regressions flow up to the peer's next benchmark refresh.
- You keep the boundary crisp — the eval team does not run MMLU sweeps or tune batching, and the peers do not sign shipping decisions.

Chapter 05 of mod-112 (the next module) picks up the program-owner posture — how the trade-off report, the routing eval, the replacement-regression harness, the app-altitude latency report, and the per-feature cost report all become load-bearing program artefacts a director / VP defends at business review.

## Summary

- The module owns the **app-altitude trade-off surface** — quality on cohorts, cost per feature, TTFT / TPOT / streaming p95 as users see them, safety in prod. It does not own **model-altitude benchmarks** (`model-evaluation-engineer`, L30) or **fleet-altitude serving** (`ai-infra-performance`).
- **Three triggers per peer** — model-altitude escalation for capability-profile gaps, distillation upstream decisions, unattributable-to-app regressions; fleet-altitude escalation for fleet-queue delta growth, self-hosted cost dominated by fleet, MLPerf-benchmark-vs-app gap.
- The **escalation ticket** is a formal artefact — triggering signal, ask, what this module hands over, what it does not, timeline. Filed in the peer's tracker, reviewed at a joint meeting.
- The **reconciliation contract** is bidirectional: peer capability profiles and MLPerf serving benchmarks flow into the app-altitude report; app-altitude cohort regressions and per-feature cost lines flow up to the peer's next benchmark refresh.
- **Outgrew vs. mis-sized** — if the fix is "hire the peer" or "the peer needs to produce this artefact," mis-sized (bootstrap with a hand-off timeline; do not absorb); if the fix is "the peer function exists and is producing artefacts on cadence," the boundary is healthy.
- **The trade-off decision stays with the eval team and product** after any escalation. Escalation transfers a specific altitude artefact, not the shipping decision.
- **Eval-team failure modes**: scope creep upward (running peer benchmarks), scope creep sideways (writing product prompts), under-scoping (report ships without reconciliation).
- **Peer failure modes**: refusing to reconcile, wanting to own the trade-off report, absorbing the cohort roster.
- **Quarterly delegation-boundary check** — six triggers reviewed, absorbed-scope reviewed, reconciliation contracts checked, signable-report cadence checked.

The module ends here. The five exercises materialise the multi-objective report, the routing eval, the replacement-regression harness, the app-altitude latency report, and the per-feature cost report. Together they are the app-altitude trade-off surface — the decision the previous ten modules have been feeding into — and the interface the peers reconcile against.
