# Delegation to Model-Evaluation-Engineer: Where the Slice Ends

## Motivation

Six chapters ago, the module declared its scope explicitly: **a product-eval slice**, not a cross-modality, multi-tenant, capacity-planned eval-as-a-service platform. That distinction is easy to hold on day one and easy to lose on day 200 — the plane works, the product teams use it, the eval-team lead's calendar fills with "can you add support for X" requests, and the drift from *product-eval slice* to *EaaS platform for the whole org* happens in three-week increments.

This chapter is the anchor. It walks:

- Where the slice ends and the peer's remit begins.
- The escalation shape — when to invite `model-evaluation-engineer` in, what to hand them, what *not* to hand them.
- The difference between "we outgrew the slice" (a legitimate escalation) and "we mis-sized the slice" (a scoping problem the eval team fixes without escalation).
- Failure modes to avoid on both sides of the boundary.

The chapter is short by intent. Its whole job is to make the boundary crisp before it becomes ambiguous under pressure.

## Core concepts

### The peer's remit at a glance

`model-evaluation-engineer` is the Model-Development-family peer at level 30. Their track owns the depth this module deliberately does not build. The scope difference in one table:

| Axis | Product-eval slice (this module) | Model-evaluation-engineer (peer) |
|---|---|---|
| **Modality coverage** | Whatever the product surface needs — usually text, sometimes text + tool traces | All modalities the org serves — image, audio, video, multimodal, embeddings; the whole raster |
| **Tenant model** | Cost-centre-per-team; soft isolation good enough for internal use | Per-tenant hardened isolation; per-tenant billing; per-tenant SLA; contractual promises |
| **Customer model** | Internal product teams; the eval-team lead is the "product manager" of the slice | Multiple internal and external paying customers; formal on-call SLA; formal roadmap |
| **Capacity planning** | Sized for the product lines the slice serves; new team joining is a review | Whole-org capacity plan; forecasts against org-wide product forecast; capacity is a contract |
| **Cost governance** | Per-team quotas; monthly reconciliation | Cross-product-line chargeback; enterprise cost centres; finance controls |
| **Rubric authorship** | Owned by product teams (with eval-team review) — mod-104 rules apply | Same |
| **Runbook authorship** | Owned by application teams and the eval team — mod-106 / mod-108 rules apply | Same, at platform scale (adds platform-level runbooks the peer owns) |
| **Runner adapter set** | Whichever runners the product teams use — 3 – 7 typical | Wider set; usually adds specialised runners for multimodal, agent trajectory, whole-benchmark-suite runs |
| **Uptime posture** | Business-hours on-call with best-effort out-of-hours | 24/7 rotation; formal error budgets against per-tenant SLA |

The peer is not "one level up." They are a different job, with different training, a different remit, and different deliverables. The transition from slice to platform is a *hand-off*, not a promotion.

### When to escalate

Three triggers, each unambiguous. If one fires, invite the peer in. If none fire, hold the slice.

**Trigger 1 — cross-modality demand from ≥ 2 product lines.** A single product line asking for image evals can sometimes be absorbed by the slice (add a runner adapter, extend the case-body schema for image inputs, wire an image-capable judge). Two product lines asking for it at the same time, and neither being solvable by extending one runner, is the peer's territory. The peer's remit already covers multimodal; asking them to build it once for the org is cheaper than the slice building it twice.

**Trigger 2 — a paying-customer contract that names an SLA the slice cannot commit to.** A tenant contract that includes "eval-run p95 latency ≤ 60 s" or "eval-run availability ≥ 99.9 %" or "per-tenant chargeback with monthly line-item invoicing" is *not* a class-A SLO adjustment. It is a per-tenant, contractually-enforced, uptime-committed guarantee — the peer's shape. The slice's SLOs are internal-customer commitments; the peer's are external. Different insurance liability, different escalation surface, different oncall.

**Trigger 3 — the capacity plan reaches the plane's structural ceiling.** Chapter 06's capacity model assumes a single-region, single-plane deploy. Once the aggregate class-C load approaches the vendor's per-region rate-limit envelope and adding more classes would starve one, the answer is a multi-region plane with per-region envelope management — which is the peer's design surface. The slice can serve ~10 – 20 mid-sized product teams before this ceiling; beyond it, escalate.

Anti-trigger: **"the slice feels ambitious."** A slice can grow substantially — more teams, more runners, more suites, more SLOs — without hitting any of the three triggers. Confusing "large slice" with "should be a platform" is the peer-scope-creep failure mode.

### The escalation ticket

When one of the three triggers fires, the escalation is a specific ticket, filed against the peer's track. Its shape:

```markdown
# Eval-data-platform delegation escalation — <trigger>

## Triggering signal
<Which of the three triggers fired, with the specific data — the product lines,
the contract, the capacity-model output.>

## Slice hand-over contract
The slice hands the peer these load-bearing artefacts (chapter 07 contract):

- **Schema.** The chapter 02 table set, the lineage key set, the reproducibility
  check, the OpenLineage integration. Repo path: <path>. Coverage: the last N
  months of runs, K TB of eval_results, M eval_sets, live and queryable.
- **Runner adapter interface.** The chapter 04 `RunnerAdapter` protocol, the
  three (or more) wired adapters, the `EvalRunSpec` / `EvalResultRow` / manifest
  contract. Repo path: <path>.
- **SLI history.** The chapter 06 dashboard, the last M months of SLI data, the
  runbooks, the postmortems. This is the peer's baseline for what "operational"
  looks like at slice scale.
- **Test-case archive.** The chapter 03 `eval_sets` + `eval_cases` archive, with
  every case's contributor, tags, versions, and history. This is the peer's
  seed corpus.
- **Cost-centre roster.** The current cost centres, quotas, pricing snapshots,
  and the last N months of invoice reconciliations.

## What the slice does *not* hand over
- **Rubric authorship.** Product-team-owned; the slice is not the rubric owner.
- **Application-specific runbooks.** mod-106 / mod-108 shipped these; they stay
  with the application teams.
- **Product ownership of any product line's eval program.** The slice serves
  those programs; the peer, once accepted, does too. The product programs
  themselves are not part of the hand-over.

## Open questions for the peer
<Explicit questions the peer needs to answer before accepting: for example,
which cross-modality runners does the peer plan to standardise on; what is the
peer's per-tenant isolation model; what is the migration path for the current
cost-centre roster.>

## Timeline
<Target hand-over date; migration steps; who owns which step; when the slice's
on-call rotation transitions to the peer's.>

## Rollback
<What happens if the peer's platform is delayed. The slice does not disappear
when the escalation is filed; it continues to operate until the peer is ready.
This section names the trigger for a rollback — "if the peer's platform is not
serving cross-modality by <date>, we hold the slice indefinitely.">
```

The ticket is a formal artefact — it lands in the peer's tracker, it is reviewed at a joint meeting, it produces a decision (accept, accept with modification, decline with reason). A slack DM that says "we should probably escalate" is not an escalation.

### "We outgrew the slice" vs "we mis-sized the slice"

Two very different situations that both feel like "we need more platform than we have."

**Outgrew.** The slice was correctly sized when it was built. Since then, the org has genuinely grown into a shape the slice cannot serve — a new product line with a new modality; a new customer with a contractual SLA; sustained aggregate throughput approaching the vendor envelope. This is a real escalation.

**Mis-sized.** The slice was under-scoped from the start. A cost-centre model that should have supported per-project was left at per-team; a schema that should have supported multi-turn was collapsed to single-turn; a class-C throughput budget that should have anticipated the online-loop growth was set too low. This is a *scoping* problem the eval team fixes without escalation.

Discriminating signals:

- If the fix is "add a column," "raise the envelope," "add a runner adapter," "tighten a quota" — mis-sized. Fix internally.
- If the fix is "add a whole modality," "add per-tenant hardening," "add a 24/7 rotation with a formal SLA," "add a multi-region plane" — outgrew. Escalate.
- If in doubt, do the smaller fix first and reassess in 60 days. Escalation is expensive on both sides; a premature escalation that is then withdrawn burns credibility.

### The peer's stance on this hand-over

The peer is not sitting waiting to absorb slices. The peer has their own roadmap, their own capacity, and their own quality bar. Two things this changes:

- **Escalate early, in prose.** A ticket that surprises the peer with "please absorb this platform in Q4" is a ticket that gets declined or delayed. A conversation two quarters ahead that asks "we might be tripping trigger 1 in Q3; what would you need from us to be ready to absorb by Q4" is what actually works.
- **Be honest about the slice's rough edges.** A hand-over that describes the slice as production-ready when the reproducibility check has been failing for six weeks is a hand-over that lands the peer in a mess. The escalation ticket is candid about SLI history — including recent burns and open postmortems.

### Where the slice continues to matter after the hand-over

Even after the peer accepts, the slice does not disappear:

- **The rubrics are still product-team-owned.** The peer serves them; they don't own them.
- **The application-specific runbooks are still application-team-owned.** They point at the peer's platform for the substrate; the application team owns the run-through.
- **The product-eval program the AI Evaluation Engineer owns is still the AI Evaluation Engineer's.** The slice was infrastructure the eval team happened to build; the eval program is the eval team's job in perpetuity.

This is the difference between "the platform moved" and "our job moved." Only the first is true. Chapter 07's clarity on this is what protects the eval team's remit after the hand-over.

### Failure modes to avoid

**On the slice side.**

- **Slice-scope creep.** The slice starts building multimodal support because "we might as well," and now the eval team is doing the peer's job with the slice's headcount. Correct: stop, escalate.
- **Slice under-scoping.** The slice ships without the reproducibility check, without cost accounting, or without SLOs, and the eval team is on-call for an unmonitored platform. Correct: prioritise chapter 02 / 05 / 06 work over new runner adapters. A working slice is small; a working slice with broken SLIs is not a slice.
- **"We handed it off; not our job." after a peer accepts.** The rubrics and the program are still the eval team's. Escalation transfers *substrate ownership*, not *program ownership*.

**On the peer side.**

- **Peer disintermediates the slice too soon.** The peer builds their platform in parallel and asks product teams to migrate before the platform serves the slice's use cases at slice quality. Correct: the escalation ticket names the acceptance criteria — the peer's platform must serve at least the slice's SLIs before migration begins.
- **Peer absorbs the slice's rubrics.** The peer is not the rubric owner; product teams are. A peer platform that takes over rubric authorship has stepped on `mod-104`'s owner (the product team's LLM Application Developer or the eval engineer working with them).
- **Peer inherits the slice's runbooks unchanged.** The application-specific runbooks stay with the application teams. The platform-level runbooks the peer needs are *derived* from the slice's postmortems and SLI history; the derived set is often smaller and cleaner.

### The delegation-boundary check the slice runs quarterly

A quick, mechanised check the eval-team lead runs at the quarterly SLO review. Four questions:

- Have any of the three triggers fired since the last review? (If yes, is there an open escalation? If no, why not?)
- Has the slice added functionality that plausibly crosses the boundary? (Multimodal, cross-tenant isolation, formal SLA promises.) If yes, was the addition reviewed against this chapter's guidance? If not, roll back or escalate.
- Is the reproducibility check still passing? (Chapter 02's SLI is *the* SLI that says the slice's substrate is trustworthy.)
- Are the current SLIs' error budgets healthy? (If sustained low, the platform is not sustainably operable; that is a signal to escalate for capacity even if the triggers have not fired.)

The check is a rubric. Running it every quarter is what keeps the boundary explicit.

## Summary

- The peer `model-evaluation-engineer` (Model-Development family, level 30) owns the **cross-modality, multi-tenant, capacity-planned, SLA-committed eval-as-a-service** surface this module deliberately does not build.
- **Three escalation triggers**: (1) cross-modality demand from ≥ 2 product lines, (2) a paying-customer contract with an SLA the slice cannot commit to, (3) capacity approaching the plane's structural ceiling.
- The **escalation ticket** is a formal artefact — triggering signal, hand-over contract (schema, adapter interface, SLI history, test-case archive, cost-centre roster), open questions, timeline, rollback. Filed in the peer's tracker, reviewed at a joint meeting.
- **Discriminating "outgrew" vs "mis-sized"** — if the fix is a column, a quota, an adapter, or a raised envelope, mis-sized; if the fix is a whole modality, hardening, 24/7 rotation, or multi-region, outgrew.
- **The eval program stays with the eval team** after the hand-over. Rubrics stay with product teams; runbooks stay with application teams. Escalation transfers substrate ownership, not program ownership.
- **Slice-side failure modes**: scope creep (building the peer's job), under-scoping (shipping without SLIs / reproducibility / cost accounting), post-hand-off abdication.
- **Peer-side failure modes**: disintermediating too soon, absorbing rubrics, inheriting runbooks unchanged.
- **Quarterly delegation-boundary check**: four questions the eval-team lead runs at the SLO review to keep the boundary explicit.

The module ends here. The five exercises materialise the schema, the test-case management surface, the runner-adapter plane, the CI + online integration with quotas, and the four platform SLIs. Together they are the product-eval slice — the substrate every prior module in this track has been assuming — and the interface the peer will absorb when the org outgrows it.
