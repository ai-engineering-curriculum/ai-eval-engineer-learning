# Why Human Review, and Where It Fits In the Eval Loop

## Motivation

By the end of mod-108 you own five automated safety scorecards, a drift monitor, a canary gate, and a tier-routing judge. Each of those mechanisms produces a backlog of cases that neither a rubric nor a classifier can close by itself. A judge that voted 0.42 on a rubric with a 0.5 threshold needs a human to tell it which side of the line this row actually sat on. An online-eval alert that fires at 02:40 on a Tuesday needs a human to read the sampled traces and decide whether a product owner should see this on Wednesday. A safety-eval item that survived the runbook because the attack is a borderline case under the written policy needs a human to arbitrate the policy row-by-row.

Human review is where those three queues converge. The work is not "someone should look at these" — the queues arrive on a cadence, the reviewers are named, the schema is versioned, the SLAs are tracked, and the output flows back into the automated machinery as calibration data, triage items, rotated gold sets, and a cost line for the program budget. When the workflow is absent or ad-hoc, three failures happen in order: judge calibration ages out, drift alerts accumulate without disposition, and safety-eval findings sit in a spreadsheet. Within a quarter the automated eval machinery is quietly running on a floor that nobody has re-measured.

This module makes human review a *product-scale system*. This chapter fixes the vocabulary — expert review, user feedback, review queue, gold set vs calibration set, inter-annotator agreement, rubric stability, SLA — names the three inbound queues from the earlier modules, names the four outbound artefacts the later modules depend on, and draws the delegation boundary to the `model-evaluation-engineer` peer that keeps this track from re-deriving methodology the peer already owns.

## Core concepts

### Two sources, one set of queues

Two production mechanisms generate human-review work. They look similar from the inside of the review workbench — rows in a table waiting for a reviewer — but they arrive through different capture paths and have different owners.

- **Expert review.** The eval team recruits a small panel of reviewers who understand the written rubric and the product context, and asks them to label rows the automated machinery flagged as uncertain, as drifting, or as safety-sensitive. The reviewer is an operator inside the eval program. The row typically comes from a judge disagreement, a drift alert, or a safety-eval survivor. Chapter 02 walks the workbench for this flow.
- **User feedback.** Real users of the product tap a thumbs-up or thumbs-down, fill in a structured form, or send a free-text report. The capture layer is the product; the feedback event rides the same instrumentation plane as a span (mod-102 chapter 04), carries the `trace_id` of the turn it rated, and lands in the eval pipeline alongside the automated scores. Chapter 03 walks the capture, dedup, and triage layer for this flow.

Both streams converge into the same labelled-row shape — the shape mod-110's eval data platform stores — so downstream consumers (the mod-104 calibration step, the mod-107 safety-reviewer dashboard, the mod-108 refusal reconciliation, the mod-111 cost model) can read from one table and not three.

The vocabulary matters because the *mitigation* is different. An expert review that flags a rubric row as miscategorised results in a rubric rewrite. A user feedback signal that flags the same row as unhelpful results in a product fix. The two overlap — a bad response is often both — but the ownership arrows are not interchangeable. Keep the source column on the row.

### The three inbound queues from earlier modules

Three queues arrive by contract into this module. Each has a named upstream chapter, a known cadence, and a known schema.

- **Judge disagreements (mod-104 chapter 05).** The tier-routing judge defers uncertain cases to a human instead of forcing a verdict. "Uncertain" is a function of the confidence of the fast tier, the disagreement between the fast and the deep tier, and the policy of the rubric (zero-tolerance rubrics have a lower deferral bar than preference rubrics). The deferred rows arrive with the rubric id, the fast-tier score, the deep-tier score (if run), the retrieval traces, and the prompt / response. Human verdicts on these rows feed back into the mod-104 calibration set.
- **Online-loop alerts the runbook cannot resolve (mod-107 chapter 03).** The drift monitor raises a per-metric, per-cohort alert; the on-call reads the sampled traces; most alerts resolve with a config change, a rollback, or a confirmed benign shift. A residual fraction — new cohort behaviour the on-call does not recognise, a rubric whose definition the drift might be exposing, a surface the runbook does not cover — escalates into this module as a labelling request. The reviewer labels the sampled rows against the rubric; the labels flow back into the online-eval dataset and (if the labels confirm a rubric gap) into the mod-104 rubric revision.
- **Safety-eval survivors (mod-108 chapter 07).** The safety-eval runbook triages findings and fixes most at the app surface (chapter 05 operating-point change, chapter 06 policy clarification). The findings that survive triage — ambiguous refusals, borderline over-refusals, injection cases where the policy is silent on whether the behaviour is harmful — escalate into this module with the OWASP / ATLAS / SAIF tag preserved. Reviewer verdicts on these rows feed back into the mod-108 refusal reconciliation and into the safety policy revision loop.

The capture format for each queue is the same shape (a span-backed row, with the upstream module's metadata attached in a namespaced field) so that the workbench, the dedup layer, and the IAA job do not branch on origin.

### The four outbound artefacts the later modules consume

Four artefacts flow out of this module. Each has a named downstream module, a known schema, and a known lineage.

- **Labelled calibration data.** Every reviewer verdict on a judge-disagreement row (and every policy-owner verdict on a safety-survivor row) is a labelled example the mod-104 quick-calibration job can consume. This is how the judge stays honest as the rubric and the product surface evolve. Cadence: continuous.
- **Owner-assigned triage items.** User-feedback rows that pass dedup and triage are routed to an owner with a severity and an SLA. The owner is the on-call engineer, the eval engineer, or the PM; the routing rule is per-surface. The mod-107 safety-reviewer dashboard reads the triage queue as a loop-health metric. Cadence: hourly to daily.
- **Rotated gold sets with sealed snapshots.** When the product surface changes — a new tool, a new policy category, a model upgrade — the gold set the offline suite runs against needs to rotate so it still reflects the surface. Chapter 05 walks the rotation; the output is a new rolling gold plus a sealed snapshot of the prior gold that the mod-106 release-gate report can still re-run bit-identically. Cadence: tied to the release calendar, typically monthly or on surface change.
- **Eval-program cost input.** The per-label cost (for both in-house expert labels and vendor-pool labels), the per-month seat cost of the workbench, and the time-to-first-review latency on the review path are three numbers the mod-111 cost / latency / quality trade-off model consumes. Chapter 06 names the trade-offs; this module reports the numbers. Cadence: monthly.

The mod-110 eval data platform stores all four artefacts as first-class tables with lineage columns; the mod-112 program-owner posture treats the full set (workbench + triage pipeline + IAA cadence + rotation playbook + vendor-vs-in-house memo) as load-bearing.

### Vocabulary this module fixes

Seven terms recur across the chapters. Fix them once here and use them consistently.

- **Expert review.** A labelling pass by a small, trained panel of reviewers against a written rubric, on rows the eval team selected. The reviewers are internal operators of the eval program or external contractors managed by the eval team. Not the same as user feedback, even when the same row could appear in both.
- **User feedback.** Signal from a user of the product, captured through a thumbs widget, a structured form, or free-text, attached to the turn the user rated. The user is not operating against a rubric. Chapter 03 walks the dedup and triage that turns this into eval signal.
- **Review queue.** A pending-items table, per surface, per source (judge-disagreement, online-alert, safety-survivor, user-feedback), with an SLA, an owner, and a backlog metric. The workbench exposes the queue to the reviewer; the triage pipeline exposes it to the on-call and the product owner.
- **Gold set vs calibration set.** A **gold set** is a sealed, policy-aligned set of rows used to report the automated suite's numbers against a fixed reference — the thing the mod-106 report runs on. A **calibration set** is a working set of labelled rows the mod-104 judge calibrates against; it rotates faster than the gold and carries confidence intervals. The two share rows but are not the same artefact. Chapter 05 formalises the rotation of the gold; the calibration set belongs to mod-104.
- **Inter-annotator agreement (IAA).** A statistic over multiple reviewers labelling the same rows against the same rubric. Cohen's kappa for two reviewers on a nominal rubric, Fleiss' kappa for three or more reviewers, Krippendorff's alpha for ordinal rubrics or when labels are missing. Chapter 04 walks the product-altitude version; the peer track owns the full-methodology version.
- **Rubric stability.** The property that different trained reviewers produce approximately the same labels on the same rows against the same rubric. A rubric that is not stable cannot anchor a judge calibration, a release-gate number, or a drift alert. Chapter 04's IAA numbers are the stability signal.
- **SLA.** Service-level agreement on the review path. Expressed as *time to first label* for a row entering the queue, plus a backlog-size cap. Each inbound queue has its own SLA (safety-survivor has the strictest; user-feedback is typically the laxest). Chapter 02 names the SLA; mod-107 reads it as a loop-health metric.

### The delegation boundary to the `model-evaluation-engineer` peer

This module is the *product-side* of human review. It stops at the product-altitude discipline the eval team actually practises at product cadence; the full-methodology discipline lives in the Model-Development-family peer track. The boundary runs through chapters 04 and 06 in particular.

- **Chapter 04 (IAA) is the explicit escalation chapter.** The product-altitude pass computes Cohen's / Fleiss' kappa and Krippendorff's alpha with a bootstrap confidence interval; it reads the interpretation thresholds as a three-bucket decision (ship the rubric, revise the rubric, escalate to peer). When the result is "escalate", the peer runs the full variance decomposition, the power analysis, and the unequal-interval ordinal analysis the product-altitude pass does not attempt. The peer returns a methodology report that this module's eval team consumes.
- **Chapter 06 (vendor vs in-house) names the vendor landscape as shape templates, not as endorsements.** The product-altitude pass picks a vendor (or a hybrid) based on cost per label, latency to first label, data-residency posture, PII handling, vertical / language coverage, operator control, and exit cost. What this module *does not* do is audit the vendor's own labelling methodology — how the vendor selects and validates its labellers, how it computes and reports IAA back to the customer, how it audits label noise. That audit is a peer-track artefact. The module consumes the peer's audit report when deciding which vendors meet the methodology bar.

Two more delegations the module respects without owning:

- **The harm-model side of safety items** belongs to the `ai-risk-engineer` peer. When chapter 03's triage flags a user-feedback row as adversary-persona-relevant rather than product-quality-relevant, the row escalates there. This module runs the queue; the peer owns the harm model.
- **The eval data platform** is mod-110. The reviewed rows, the triage items, the IAA reports, and the gold snapshots are first-class tables with full lineage — but the schema and the storage are mod-110's artefact, not this module's.

Keep the boundaries clean and the eval program stays small, auditable, and defensible to the product owner. Blur them and the module either re-derives peer methodology (expensive, and the peer will not accept your version as authoritative) or quietly stops labelling because the SLA slipped and nobody owned the backlog.

### Why this module is the half of the program that keeps the automated half honest

Three grounded reasons the automated eval machinery needs the human-review half to exist.

- **Judges age out when the rubric or the surface changes.** An LLM-as-judge rubric is calibrated against a labelled set at one point in time; the model behind the judge updates, the written rubric revises, the product surface grows a new tool. Without a steady trickle of expert labels flowing in, the calibration drifts quietly. Mod-104 chapter 04 defined the quick-calibration cadence; the labels have to come from somewhere. This module is where.
- **Drift alerts without disposition are noise.** The mod-107 drift monitor fires on a statistical shift. Most shifts are benign; a fraction are real regressions. The on-call cannot distinguish by eye on ambiguous cases. When the on-call cannot resolve the alert, the row goes to this module for a labelling verdict. The disposition rate (fraction of alerts resolved by a reviewer within the SLA) is the loop-health metric mod-107 reads back.
- **Safety-eval findings need an owner that is not the eval team.** The surviving items from the mod-108 runbook are policy-adjacent, not engineering-adjacent. The policy owner (a trust-and-safety lead, a product owner, sometimes a legal reviewer) is the authority; the eval team runs the queue that puts the row in front of the right owner with the right framework tag.

Without these three loops, the automated eval runs on a snapshot — a snapshot that is correct at merge time and slowly stops being correct afterwards.

### What this module is not

- **A methodology course on inter-rater reliability.** The theory — variance decomposition, power analysis, prevalence and bias correction, the full ordinal treatment — is peer-track content. This module computes enough to ship a rubric or escalate.
- **A benchmark-construction course.** Building a benchmark the field reads as authoritative (held-out test sets, published task coverage, provenance audit) is model-eval work. This module rotates a *product* gold on a *product* cadence.
- **An LLM-as-judge course.** mod-104 owns that end-to-end. This module *feeds* it labels and *consumes* its deferrals; it does not re-derive the judge.
- **A harm-model course.** The `ai-risk-engineer` peer owns adversary personas, red-team data generation, hazard categorisation. This module runs the review queue those artefacts flow through, not the artefacts themselves.
- **A vendor-management course.** Procurement, contract negotiation, DPA review, and vendor renewal cadence are a program-operations function. Chapter 06 is a shape template for the technical trade-offs; the paperwork is elsewhere.
- **An annotation-UI course.** The workbench this module stands up is Argilla or Label Studio *as a tool* — configured, wired to the trace store, and operated. The internals of either tool are their own documentation.

### The end-of-module posture

By the end of this module you own the human-in-the-loop half of the eval program. Concretely, you can:

- Stand up an Argilla or Label Studio workbench wired to the mod-102 trace store with lineage columns on every reviewed row.
- Capture, deduplicate, and triage user feedback into an eval dataset with per-surface owner, severity, and SLA.
- Compute Cohen's / Fleiss' kappa and Krippendorff's alpha on a reviewed set with a bootstrap CI and interpret the result as ship / revise / escalate.
- Rotate the gold set when the product surface changes and keep a sealed snapshot that re-runs the old mod-106 report bit-identically.
- Pick vendor, in-house, or hybrid for a given surface against a trade-off matrix and defend the pick to the product owner and the program owner.

Chapter 02 starts with the expert-review workbench — the schema, the trace-store integration, and the rota.

## Summary

- Human review is the half of the eval program that keeps the automated half honest. Without it, judge calibration ages out, drift alerts accumulate without disposition, safety-eval findings sit in a spreadsheet.
- **Two sources** feed the queues: **expert review** (operators of the eval program labelling against a rubric) and **user feedback** (product users rating the turn they just saw). Same shape, different owners.
- **Three inbound queues** arrive by contract: judge disagreements from mod-104, online-loop alerts the runbook cannot resolve from mod-107, safety-eval survivors with OWASP / ATLAS / SAIF tags from mod-108.
- **Four outbound artefacts** flow downstream: labelled calibration data to mod-104, owner-assigned triage items to mod-107, rotated gold sets with sealed snapshots to mod-106 / mod-110, cost input to mod-111.
- **Seven vocabulary terms** recur: expert review, user feedback, review queue, gold set vs calibration set, inter-annotator agreement, rubric stability, SLA.
- **Delegation to `model-evaluation-engineer`** covers full IAA methodology (chapter 04 escalation) and vendor-methodology audits (chapter 06); delegation to `ai-risk-engineer` covers harm-model-side review; mod-110 owns the storage substrate.

Chapter 02 walks the expert-review workbench — Argilla or Label Studio, wired to the trace store, operated with a rota and an SLA.
