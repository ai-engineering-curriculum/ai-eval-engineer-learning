# exercise-02: Offline vs Online vs Deferred

**Estimated effort:** 2 hours

## Objective

Take the criteria table you produced in exercise-01 and place every row into one of three buckets: **offline regression suite**, **online sampled-trace eval**, or **deferred to a named peer role**. Justify each placement with the four-step decision rule from chapter 03, and build the sampling and cost budgets that make the plan operable.

## Prerequisites

- Exercise-01 completed for at least one surface.
- Chapter 03 of this module.
- A rough sense of your production-traffic order of magnitude (10s, 100s, 1000s, 10k+ interactions per day). Use a plausible number for your chosen surface if you do not have one.

## Requirements

Produce a single markdown file `placement-table-<surface-slug>.md` extending your exercise-01 table with three new columns and three supporting sections.

### 1. Placement table

Add these columns to your criteria table:

- `placement` — `offline` / `online` / `both` / `deferred`
- `decision-rule-step` — the step from chapter 03's decision rule that landed the placement (`1: defer`, `2: offline ship gate`, `3: online only`, `4: both`).
- `deferred-to` — peer role name and the interface, when applicable (e.g., `model-evaluation-engineer — calibration methodology packet`).

### 2. Offline suite design (short — one page)

For every row placed offline or both:

- **Fixture sources.** Split the fixture set into author-crafted, snapshot-replay, and adapted public-benchmark fixtures. Give order-of-magnitude counts per criterion.
- **Judge budget per CI run.** Estimate judge cost for one full run of the offline suite (fixture count × per-item judge cost). If the total is over $5 per PR, redesign until it is not.
- **Merge gate vs canary gate.** For each offline SLO, which class of gate?
- **Pinned versions.** Which of `model`, `prompt`, `retriever`, `decoding`, `seed`, `fixture-hash` you would pin, and where you would store the manifest.

### 3. Online loop design (short — one page)

For every row placed online or both:

- **Sampling policy.** Rate (percentage of traffic), stratification (by cohort / feature flag / traffic source), and per-family cap. Defend the choice against uniform-random sampling.
- **Reporting window and cadence.** Rolling window per SLI, dashboard update cadence, alert page cadence.
- **Judge / rule stack.** Which SLI uses which scorer type.
- **Judge cost budget.** Monthly $ ceiling. Reconcile with sampling rate and traffic volume — show the arithmetic.
- **Alerts and on-call.** Which SLI wakes whom.

### 4. Deferred handoff table

For every row placed as `deferred`, one row in a table with columns: `criterion`, `deferred-to (peer role)`, `interface artefact`, `what you rely on them to provide`, `how you consume it`.

At minimum:

- **Judge-vs-human calibration methodology** for any judge-scored SLI → `model-evaluation-engineer`.
- **Harm model authoring** if you have a safety criterion → `ai-risk-engineer`.
- **Audit trail / regulator interface** if the surface is in a regulated domain → `ai-evaluation-engineer` (Governance family, level 25).

If you have nothing to defer, that is a **finding** — say why the plan is complete without any peer methodology.

## Starter guidance

- Walk the four-step decision rule in order for each row. Do not skip step 1.
- The "kitchen sink" anti-pattern lives here — resist the urge to put every row in `both`. If a criterion has no online value, do not run it online.
- The **cost budget** is what makes the placement real. If your online judge budget is $10k/month, you have not designed a plan; you have designed a bill.
- Uniform-random sampling is almost never right. Stratify by at least one dimension.
- Reserve merge-gate status for SLIs where a change under review can *plausibly* fail them. Merge-gating an SLI that never fails is CI theatre.

## Acceptance criteria

You are done when:

- Every row of the criteria table has a placement, a decision-rule step, and (if applicable) a peer handoff.
- The offline suite section has fixture-count estimates and a defensible per-PR cost.
- The online loop section has sampling policy, cost arithmetic, and an on-call assignment per alerting SLI.
- The deferred handoff table names at least one peer role, or the plan explains its absence.
- The whole file, together with exercise-01's output, could be handed to an ML engineer to implement offline and to an eval-platform engineer to implement online.

## Stretch goals

- **Sensitivity analysis on the sampling rate.** For one online SLI, compute what sampling rate gives you a detectable weekly regression at plausible effect size, using the bootstrap intuition (depth in `mod-107`). Note where you would want `model-evaluation-engineer` to check the arithmetic.
- **Shadow-launch plan.** For one criterion, sketch how you would run a shadow-eval on the next model bump before shipping (interfaces into `mod-107`).
- **Cost report.** Produce a one-page cost report for the plan: monthly offline CI $, monthly online judge $, and the criterion each dollar buys.

## What this exercise does *not* cover

You are not implementing the offline harness yet (`mod-106`), the online loop yet (`mod-107`), or the sampling methodology depth (`model-evaluation-engineer`). You are producing the design a subsequent module and a peer track will implement.
