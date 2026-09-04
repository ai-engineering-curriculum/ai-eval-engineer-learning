# Release-Gate Architecture for One App Family: Roadmap → Gates → Thresholds → Rollback

## Motivation

Chapter 01 named the *ad-hoc gate* failure shape: every release argues its own bar, and six months in the org cannot answer "what does 'ready to ship' mean for surfaces in this family?" This chapter builds the artefact that answers the question — a **release-gate architecture** for one app surface family, published as document + code, mapped to every surface in scope, and binding on every release the family runs.

The chapter is deliberately scoped to *one* family. The reason is straightforward: a release-gate architecture that tries to cover every surface in one org-wide policy either becomes a lowest-common-denominator wall that nothing meaningful clears, or a matrix so parameterised that no one can read it in a sitting. The right unit is the family — the grouping of surfaces that share an evaluation profile (an internal knowledge-assistant family, a customer-facing support-agent family, a developer-copilot code family). One architecture per family; families are stood up one at a time; policies are refreshed on the quarterly cadence chapter 01 named.

Two symptoms show a family lacks a gate architecture:

- **The release checklist has no `eval evidence` field.** The team ships when the code review is approved and the on-call is available. The eval team's numbers, if produced, are consulted; they do not gate. When a regression happens, the retrospective has no "what did the gate say?" row because there was no gate.
- **The gate is a Slack message.** The eval team runs a scoring pass, posts "looks good, go ahead" in a release channel, and the release proceeds. There is no thresholded pass/fail, no artefact hash, no signer. The signal is opinion, not evidence.

Both symptoms make the eval program un-defensible when a regression ships. This chapter closes them.

## Core concepts

### A gate is not a check; it is a *contract*

Software CI has a long tradition of gates — "unit tests pass or the merge blocks." The eval-program gate is the same shape, applied to a different evidence set. Three properties matter:

- **A specific artefact is required.** Not "some eval results"; a named artefact — a mod-106 replay bundle result, a mod-108 safety report, a mod-111 multi-objective report — hashed and stored.
- **A specific threshold is bound.** Not "quality looks good"; a named threshold — "faithfulness rubric mean ≥ 4.1 on the `regulated` cohort AND no cohort drops more than 0.8 rubric points versus the previous release."
- **A specific outcome is enforced.** Not "the team decides at the release meeting"; a named outcome — the release proceeds automatically if the gate passes, is held automatically if the gate fails, and is escalated to a named human if the gate is borderline (defined per-gate).

If any of the three is missing, the "gate" is a consultation. Consultations are useful; they are not gates. The distinction matters because a program cannot be defended on consultations — the audit walk-through in chapter 07 requires a documented contract per gate, not a Slack thread.

### The four-gate topology per family

The families this module addresses ship on the shape most modern LLM app releases follow: offline change → canary traffic → progressive ramp → in-production ownership. Four gates, one per stage.

```
   +-------------+     +-------------+     +-------------+     +-------------+
   |   OFFLINE   | --> |   CANARY    | --> |    RAMP     | --> |  POST-SHIP  |
   |    GATE     |     |    GATE     |     |    GATE     |     |    GATE     |
   +-------------+     +-------------+     +-------------+     +-------------+
   Evidence: mod-106  Evidence: mod-107  Evidence: mod-107  Evidence: mod-107
   replay bundle       online loop on     online loop on     online loop on
   result + mod-108    canary cohort       ramp cohorts       full production
   safety report +     + mod-111 trade-    + mod-108 safety   + monthly card
   mod-111 report      off delta           in-prod findings   slice diff
   
   Fires:              Fires:              Fires:              Fires:
   pre-merge            5-10% ramp for     graduated ramps    continuous;
   (blocks CI)          time-boxed         (25/50/100%)       triggers
                        canary window                          rollback and
                                                               investment
```

Each gate has one job.

- **Offline gate.** Runs on the CI machinery mod-106 built. Evidence: a replay-bundle result against the frozen eval set, a safety report from mod-108, a mod-111 multi-objective report for candidate-vs-incumbent. Thresholds: rubric floors, no-regression-versus-incumbent bounds, cohort-preservation contracts. Outcome: PR blocks if any threshold fails.
- **Canary gate.** Runs on the online-loop machinery mod-107 built, on a small percentage of live traffic (typically 5 – 10 %) for a time-boxed window (typically 4 – 24 h, depending on traffic volume). Evidence: online-loop rubric scores, drift signals, guardrail-trigger rates, cohort deltas. Thresholds: online-quality floor per cohort, drift alarm rates, guardrail-trigger rate bounded. Outcome: promotes to ramp if passing, auto-rolls-back if failing, escalates to on-call if borderline.
- **Ramp gate.** Runs on the same machinery, at each ramp step (25 %, 50 %, 100 % is a common shape). Evidence: the canary evidence at ramp scale, plus mod-108 in-prod safety findings, plus cost / latency deltas at ramp scale. Thresholds: same as canary tightened for statistical power, plus cost / latency bounds from the mod-111 pre-registration. Outcome: advances to next ramp step if passing, holds and investigates if failing.
- **Post-ship gate.** Runs continuously against the in-production traffic. Evidence: mod-107 online loop, mod-108 safety in-prod findings, monthly card-slice diff. Thresholds: the pre-registered *rollback triggers* from mod-111; the incident investment ledger's regression fixtures (chapter 05). Outcome: fires the rollback trigger automatically if a threshold breaks; enqueues an investment item if a fixture regresses.

The four gates share a shape (evidence + thresholds + outcome) but differ in timing and enforcement. The offline gate is a *hard* block (no merge, no exception without a signed override). The canary and ramp gates are *soft* blocks (auto-rollback + escalation). The post-ship gate is a *rolling* enforcement (continuous; triggers action rather than blocking a discrete transition).

### The gate map per surface

Not every surface in a family runs every gate on every release. A minor prompt tweak on a low-traffic surface does not warrant a 24-hour canary; a model-family swap on a high-traffic surface warrants an aggressive canary and a slow ramp. The **gate map** per surface says which gates fire for which change classes.

Example gate map for a support-agent family:

```yaml
family: customer_facing_support_agents
surfaces:
  - name: support_bot
    change_classes:
      prompt_minor:      { offline: yes, canary: 4h/5%,   ramp: 25/50/100 by 24h, post_ship: yes }
      prompt_major:      { offline: yes, canary: 8h/5%,   ramp: 10/25/50/100 by 48h, post_ship: yes }
      model_snapshot:    { offline: yes, canary: 24h/5%,  ramp: 5/25/50/100 by 72h, post_ship: yes }
      model_family:      { offline: yes, canary: 48h/5%,  ramp: 5/10/25/50/100 by 5d, post_ship: yes }
      retriever_index:   { offline: yes, canary: 24h/5%,  ramp: 25/50/100 by 48h, post_ship: yes }
      tool_added:        { offline: yes, canary: 24h/5%,  ramp: 10/25/50/100 by 48h, post_ship: yes }
      guardrail_change:  { offline: yes, canary: 12h/5%,  ramp: 25/50/100 by 24h, post_ship: yes }
  - name: internal_support_lookups
    change_classes:
      prompt_minor:      { offline: yes, canary: no,       ramp: no,               post_ship: yes }
      # ... low-traffic surface; single-step ramp is acceptable
```

Two things the map does:

- **Names the change classes.** A change class is the smallest unit of change the release-gate policy differentiates on. The seven above (prompt minor / major, model snapshot, model family, retriever index, tool added, guardrail change) cover most real-world app surfaces. Chapter 01 of mod-110 covered the lineage-key set that pins each; the change class is the human-readable label for a change-of-shape that the eval gate will treat differently.
- **Pins the ramp shape per change.** A model-family swap on a high-traffic surface gets a five-step ramp over five days; a prompt tweak gets a three-step ramp over one day. The shapes are pre-registered; they are not chosen at release-time.

The gate map is the artefact that lives with the policy. Every release names its change class; the machinery reads the map and enforces the correct shape.

### Thresholds — where they come from

The single biggest failure mode in gate-threshold-setting is thresholds pulled out of the air ("let's say 4.1 rubric points"). Real thresholds have provenance. Four sources:

- **Incumbent-relative thresholds.** "No cohort's quality drops by more than δ against the incumbent" is the most defensible shape. It is bounded by the current state, not by a theoretical ceiling; a bug in the rubric affects both sides equally; incumbents drift, so the threshold auto-adjusts. Chapter 03 of mod-111 formalises the *cohort-preservation contract* — this is the offline gate's row-shape.
- **Product-committed thresholds.** "Faithfulness rubric mean ≥ 4.1 for the `regulated` cohort" only makes sense if product has committed that 4.1 is the number the surface's product promise depends on. The threshold's provenance is the product commitment; the eval team does not invent it in isolation.
- **Safety-mandated thresholds.** OWASP LLM Top-10 category regressions, jailbreak-resistance floors, PII-leak rate ceilings. These come from the mod-108 safety report and the risk-eng peer's harm model (chapter 03's delegation contract). Non-negotiable at the gate level.
- **Regulatory-derived thresholds.** For surfaces subject to sector regulation (medical, financial, high-risk under EU AI Act), specific accuracy / robustness / cybersecurity thresholds come from the governance peer via the framework mapping (chapter 07). Non-negotiable; the governance peer signs off.

The gate policy names the provenance for every threshold. "Faithfulness ≥ 4.1 (product commitment 2026-Q2, doc `PR-Support-2026-Q2.md`)" is a provenanced threshold. "Faithfulness ≥ 4.1" is not.

### Rollback criteria — the paired half of every threshold

Every threshold has a paired **rollback criterion** — the condition that, if observed post-ship, reverses the release automatically. Mod-111 chapter 02 introduced this at the trade-off-report level; the gate architecture at program altitude makes it the *policy default*.

Shape:

```yaml
rollback_criteria:
  - name: faithfulness_regulated_cohort
    metric: online.faithfulness.mean
    cohort: regulated
    window: 24h
    threshold: baseline - 0.5
    action: flag_flip
    flag: support_bot_v42_ramp
    grace_period: 24h_since_previous_rollback
    runbook: runbooks/support-bot-faithfulness-rollback.md
    escalation: eng_lead + product_owner + eval_program_owner
    
  - name: safety_jailbreak_rate
    metric: online.safety.jailbreak_success_rate
    cohort: all
    window: 6h
    threshold: baseline + 0.5pp   # more than half a percentage point up
    action: flag_flip + on_call_page
    flag: support_bot_v42_ramp
    grace_period: none            # no grace on safety
    runbook: runbooks/support-bot-jailbreak-rollback.md
    escalation: risk_eng + ai_infra_security + on_call
```

Three properties matter:

- **Bound to a specific mod-107 metric.** Not "quality looks bad"; a named online-loop metric with a cohort filter and a window. Chapter 04 of mod-107 covered these; the gate policy references them by name.
- **Bound to a specific action.** A flag flip (the common case), a router-tier restriction, an A/B ramp-down. Not "we page someone" — pages are the *notification*, not the action.
- **Bound to a specific runbook.** The runbook covers "what to do after the rollback fires" — root-cause investigation, communication to product, incident-review scheduling, investment-ledger entry (chapter 05). No rollback trigger ships without a runbook.

Safety rollbacks typically have *no grace period* (any breach fires immediately); quality rollbacks typically have a grace period (the first firing in a 7-day window fires automatically; a second firing requires a human confirm to prevent oscillation). The policy names both.

### The release-runbook binding

The gate policy names the artefacts, thresholds, and rollback criteria; the **release runbook** names the *procedure*. Mod-106 chapter 03 built the release-runbook shape at the practitioner level; the program-owner policy makes it a family-wide binding.

Per surface, the runbook covers:

- **Pre-release.** Change class classification; gate map lookup; evidence-bundle preparation; pre-registration filing (mod-111 chapter 02).
- **Offline gate.** Run the mod-106 replay bundle; check thresholds; get sign-off from named signer (usually eng lead + eval-owner).
- **Canary gate.** Enable canary flag; monitor the mod-107 canary dashboard for the pre-registered window; auto-promote if passing.
- **Ramp gate.** Advance ramp; monitor at each step; hold and investigate on threshold breach.
- **Post-ship gate.** Confirm rollback triggers armed; confirm alert routing correct; confirm investment-ledger entry created for any tuning needed.
- **Rollback path.** Explicit steps to fire each rollback trigger manually; escalation contacts; incident-declaration criteria; investment-ledger entry after resolution.

The runbook lives with the family policy in the same repo; the release machinery reads it. A release that starts with "let me remember what we do" is a release with no runbook binding.

### Policy composition — one document per family

The policy for a family is one document. Concretely, a Markdown or YAML file in the eval-program repo, versioned, signed, and referenced by the CI and canary machinery. A reasonable shape:

```
programs/gates/customer_facing_support_agents/
├── policy.yaml                       # the gate policy — surfaces, gate map, thresholds
├── rollback_criteria.yaml            # rollback triggers + runbooks (referenced by policy.yaml)
├── change_classes.md                 # what "prompt minor" vs "prompt major" means for this family
├── evidence_shape.md                 # what artefacts each gate reads; where they live in mod-110
├── runbooks/
│   ├── support-bot-release.md
│   ├── support-bot-faithfulness-rollback.md
│   └── support-bot-jailbreak-rollback.md
├── history/
│   ├── 2026-Q2-refresh.md            # quarterly refresh notes with threshold changes
│   └── 2026-Q3-refresh.md
└── README.md                         # signers, cadence, escalation contacts
```

The document has one signer per role — the eval-program owner, the product owner for the family, the engineering lead for the family, the safety peer (`ai-risk-engineer` or the governance peer, per the org's shape), and the governance peer for surfaces in regulated scope. Missing signatures block a policy refresh from taking effect.

### The quarterly refresh — how thresholds move

Thresholds move over time. A product-committed threshold that was "faithfulness ≥ 3.8" a year ago and now sits at "faithfulness ≥ 4.2" reflects a maturing surface. A rollback criterion that fires monthly indicates either a real quality drift or a threshold that is too tight for the current baseline.

The quarterly refresh is where movement is authored. Two things the refresh does:

- **Reviews every threshold against last quarter's evidence.** Did the offline gate block anything meaningful? Were the rollbacks firing appropriately (not too often, not too rarely)? Are the cohort-preservation deltas still tight enough for the surface's product commitments? A threshold that never fires might be redundant; a threshold that fires weekly might be miscalibrated.
- **Absorbs any new change classes.** If a new tool was added, does the `tool_added` class still fit or does it need a new class? If a new surface joined the family, is its gate map defined? If a new safety category (mod-108 chapter 05) landed from the risk-eng peer's harm-model refresh, is it in the offline gate?

The refresh is a document (a Markdown page in `history/`) with the old thresholds, the new thresholds, the evidence citations, and the signer set. The refresh is what makes the policy a *living* artefact rather than a founding document that ossifies.

### Exceptions — the discipline the policy needs

Every real release-gate policy will have edge cases. A vendor snapshot bumped in the middle of an incident; a surface running a customer-committed pilot on a shortened timeline; a low-risk change class where the canary window would delay a bugfix that has to ship today. The policy needs an **exception mechanism** — otherwise the exception becomes "the policy is wrong, ignore it" and the policy erodes.

Two shapes:

- **Documented waiver.** A specific gate is waived for a specific release, with a specific reason, a specific signer (usually the program owner + product owner + eng lead), a specific expiry, and a follow-up commitment (usually an investment-ledger entry). The waiver is filed in `history/waivers/`; the release machinery accepts it as a signed override.
- **Emergency path.** A named "emergency" release class that runs a subset of gates (offline mandatory; canary skipped; post-ship gate armed on shorter windows). Emergency-class releases are counted; if the count exceeds a threshold (say, 2 per quarter for a family), the policy triggers a refresh.

Neither shape is "ship without a gate." Both keep the gate accountable — the waiver is a documented artefact reviewed at the quarterly refresh; the emergency-class is a named path with tighter post-ship monitoring.

### Anti-patterns to avoid

**"One org-wide policy."** A single policy trying to cover every app surface family becomes a lowest-common-denominator wall (nothing meaningful clears it) or a matrix so parameterised no one reads it. Fix: one policy per family; families are stood up one at a time.

**"Thresholds without provenance."** "Faithfulness ≥ 4.1" with no source. When the number is questioned six months later, no one can defend it, and it silently moves to whatever passes today. Fix: every threshold names its provenance (incumbent-relative, product-committed, safety-mandated, or regulator-derived).

**"Rollback triggers without runbooks."** The rollback fires; the on-call has no procedure; the release-related-cascade begins. Fix: no rollback trigger ships without a paired runbook.

**"The gate is the eval team's problem."** Eval publishes numbers; the release goes ahead if the release meeting mood is positive. Fix: the gate is bound in the CI / canary / ramp machinery — the release cannot advance without a passing artefact. The eval team publishes evidence; the machinery enforces.

**"No exception mechanism."** Every real edge case erodes the policy because "the policy is wrong." Fix: documented waivers with signers and expiry; a named emergency path with tighter monitoring.

**"Policy set once, never refreshed."** Thresholds ossify; drift is unaccounted; the policy stops matching reality. Fix: quarterly refresh with a documented `history/` trail.

### A minimal walkthrough

To make the shape concrete, an end-to-end walkthrough for a single release under an existing family policy.

A prompt-major change is landing on the `support_bot` surface. The engineer's PR triggers the release machinery.

**Change class classification.** The PR labels itself `change_class: prompt_major` per the change-classes doc. The machinery looks up the gate map for `support_bot` and `prompt_major`: offline + 8h/5% canary + 10/25/50/100 by 48h ramp + post-ship gate.

**Offline gate.** The CI job pulls the frozen eval set (`support-eval-v14`), runs the mod-106 replay bundle against the new prompt, generates the mod-108 safety report, and builds the mod-111 multi-objective report against the incumbent. Thresholds: no cohort drops > 0.8; cost per request ≤ incumbent + 15 %; no OWASP LLM Top-10 category regressions. All three pass. The engineering lead + eval-owner sign the report. The PR merges.

**Canary gate.** On merge, the canary flag flips for 5 % of traffic. The mod-107 dashboard shows a per-cohort quality track for 8 hours. At the 8-hour mark, no cohort's online-loop faithfulness rubric has dropped more than 0.3 points; the guardrail-trigger rate is within the pre-registered band; the policy-violation rate is unchanged. The machinery promotes to the first ramp step (10 %) automatically.

**Ramp gate.** Every ramp step (10 %, 25 %, 50 %, 100 %) waits until the online-loop metrics have stabilised for the pre-registered dwell (typically 6 – 12 h). Each transition is auto-promoted if the thresholds hold. During the 25 % → 50 % transition, the mod-107 dashboard shows the `regulated` cohort's faithfulness drop by 0.4 points over 4 hours. The threshold (drop > 0.5 over 24 h) is not breached yet, but the trend has crossed the *warning* threshold (drop > 0.3 over 4 h). The machinery holds the ramp at 25 %, pages the on-call, and links the release runbook.

The on-call reviews the trend against the change-class expectation ("prompt major on `regulated` cohort — some drift is expected within Δ = 0.5"). The drift is within the tolerance; the on-call annotates the release with the observation and manually promotes to 50 %. At 100 %, the trend has re-stabilised.

**Post-ship gate.** The rollback triggers arm. The `faithfulness_regulated_cohort` trigger monitors the online-loop mean on a 24-hour window; the `safety_jailbreak_rate` trigger monitors the safety guardrail on a 6-hour window. Both are quiet through the first week.

**Retrospective.** The release meets at the monthly review. The eval-program owner reviews the "release advanced from canary to 100 % in 52 hours; one manual hold at 25→50; no rollbacks fired." The change is annotated as *baseline* for the next `prompt_major` release on this surface — the 25→50 hold is a normal event, not an escalation.

At no point does anyone at the release meeting negotiate what "ready to ship" means. The gate policy said. The evidence was produced. The machinery enforced. The retrospective reviews the pattern, not the policy.

### The end-of-chapter posture

By the end of this chapter you can:

- Author a **release-gate architecture** for one app surface family — surfaces in scope, gate map per surface × change class, thresholds with provenance per gate, rollback criteria per threshold, runbooks per rollback.
- Bind the architecture to the **CI / canary / ramp / post-ship machinery** so the gates are *enforced*, not consulted.
- Publish the policy as a versioned, signed document with a **quarterly refresh cadence** and a documented **waiver / emergency-path mechanism**.
- Compose the gate outcomes into the **investment ledger** (chapter 05) and the **card slice** (chapter 06) so the program's downstream artefacts read from the gate history rather than from Slack scrollback.

Chapter 03 picks up the delegation-contract set — the mechanism that keeps the peer specialists producing what the gate needs without absorbing eval scope.

## Summary

- The chapter builds one **release-gate architecture** for one **app surface family**. One policy per family; families are stood up one at a time; policies are refreshed quarterly.
- A **gate** is not a check; it is a contract with three properties — specific artefact, specific threshold, specific outcome. If any is missing, it is a consultation.
- Four gates per family: **offline** (blocks the merge, evidence from mod-106 + mod-108 + mod-111), **canary** (5 – 10 % traffic for hours, evidence from mod-107), **ramp** (staged 25/50/100 % over hours-to-days, same evidence), **post-ship** (continuous; fires the rollback triggers).
- The **gate map** per surface × change class pins which gates fire in what shape for what change. Change classes named up front (prompt minor / major, model snapshot, model family, retriever index, tool added, guardrail change).
- **Thresholds have provenance** — incumbent-relative, product-committed, safety-mandated, or regulator-derived. Every threshold names its source.
- Every threshold has a **paired rollback criterion** — bound to a mod-107 metric, an automatic action, a runbook, and (for quality) a grace period.
- The policy is one **versioned, signed document** per family; the release runbooks live with it; the CI / canary / ramp machinery reads it.
- **Quarterly refresh** authors threshold movement, absorbs new change classes, and reviews rollback firing rates against the last quarter's evidence.
- **Exceptions are a discipline** — documented waivers with signers and expiry; a named emergency path with tighter post-ship monitoring — not "ignore the policy."
- **Anti-patterns**: one org-wide policy, thresholds without provenance, rollback triggers without runbooks, gate-as-consultation, no exception mechanism, policy set once and never refreshed.

Chapter 03 opens the delegation contracts — the shape that keeps the gate's evidence base sourced from the right peer specialists without absorbing peer scope.
