# Governance Mapping: NIST AI RMF, EU AI Act, ISO/IEC 42001 — What the Eval Program Owes

## Motivation

Every prior chapter built an artefact — a release-gate policy, a delegation-contract set, a build-vs-buy platform matrix, an incident investment ledger, a per-family card slice. Each is defensible internally. This chapter puts the artefacts in front of an external framework — NIST AI RMF, EU AI Act, ISO/IEC 42001 — and asks: *do the frameworks' requirements find the artefacts they need in this eval program?*

The question is not academic. Orgs shipping AI at scale are increasingly asked (by internal risk committees, third-party auditors, customer procurement teams, and regulators) to demonstrate framework alignment. "Show me your NIST AI RMF *Measure* function evidence for this surface." "Where is your EU AI Act Article 15 accuracy and robustness evidence?" "Which ISO/IEC 42001 controls does your evaluation activity satisfy?" The audit-walkthrough happens; the demonstration is expected in a single sitting.

The eval program does not *own* framework compliance — that is the governance peer's remit (`ai-evaluation-engineer` at L25 for release-assurance framing, `ai-governance-analyst` for org-wide policy). The eval program *does* own the **framework mapping** — the crosswalk between the program's artefacts (chapters 02 – 06) and the specific framework provisions the artefacts evidence. The mapping is what makes the audit walkthrough tractable. Without it, the auditor walks through the eval program's dashboards, the governance peer paraphrases, and the mapping is reconstructed on the fly with the risk of gaps.

Two symptoms show the mapping is missing:

- **The audit walkthrough is a workshop.** The auditor asks for evidence against a specific NIST AI RMF action; the governance peer huddles with the eval team for an hour to figure out which artefact answers. The walkthrough turns into an evidence-hunt.
- **The framework refreshes catch the program by surprise.** NIST publishes a Playbook update; the EU AI Act's harmonised standards get published; ISO/IEC 42001 revises. Nobody has been tracking; the program has to scramble to update.

Both symptoms are integration failures. This chapter closes them.

## Core concepts

### The three frameworks in scope — and the peer boundary

The frameworks the program maps against depend on the org's regulatory posture, customer set, and geographic footprint. Three are named in the module's learning objectives; the mapping shape generalises to others.

- **NIST AI RMF (AI 100-1) and the Generative AI Profile (AI 600-1).** United States, voluntary framework, widely adopted as a de-facto structure by orgs building internal AI governance. Four functions (Govern, Map, Measure, Manage); the eval program's artefacts live primarily under **Measure** and **Manage**. The Generative AI Profile adds GenAI-specific actions layered on top of the base functions.
- **EU AI Act (Regulation (EU) 2024/1689).** European Union, binding regulation on AI systems placed on the EU market. High-risk system obligations (Article 9 risk management, Article 10 data governance, Article 12 record-keeping, Article 15 accuracy / robustness / cybersecurity, Article 61 post-market monitoring, Article 72 post-market monitoring plan, Article 73 serious-incident reporting). General-purpose AI (GPAI) model provider obligations (Articles 51 – 55 and their annexes). The eval program's artefacts feed the technical-documentation, testing, and post-market monitoring obligations.
- **ISO/IEC 42001:2023 — AI management systems.** International standard for an AI management system (analogous to ISO/IEC 27001 for information security). Controls cover organisational context, planning, support, operation, performance evaluation, and improvement. The eval program's artefacts feed the operational and performance-evaluation control clauses.

<!-- needs-research: on the next research cycle, re-verify the current status of the EU AI Act GPAI provider obligations (Articles 51–55 came into force in phases with obligations attaching from 2025-08-02 for new models and 2027-08-02 for models already on the market; the Commission's Code of Practice for GPAI providers may be in force by now), the NIST AI RMF Playbook action set (revisions publish periodically), and ISO/IEC 42001:2023 (the standard is stable but the guidance ecosystem evolves). Refresh chapter 07 and exercise-05 accordingly. -->

The **governance peer** owns the *interpretation* — which framework provisions apply to which surfaces; what "high-risk" means in the org's specific product context under EU AI Act Annex III; how the org's ISO/IEC 42001 audit scope is drawn. The **eval program** owns the *evidence base* — for every provision the governance peer says applies, the program's artefacts either evidence it or the peer authors an escalation for the gap.

The mapping is authored jointly. The chapter's exercise-04 target has the reader author the mapping for one family; the exercise runs against the peers' current interpretation.

### The mapping's shape — one row per artefact-to-provision link

The mapping is a document (or a YAML table) with one row per (program artefact, framework provision) pair. The row names the artefact, the provision, the shape of the evidence the artefact provides, and any residual gaps.

Example rows (excerpts):

```yaml
mapping:
  # NIST AI RMF Measure function
  - artefact: release_gate_policy
    artefact_ref: programs/gates/customer_facing_support_agents/policy.yaml
    framework: nist_ai_rmf
    provision: MEASURE 2.1
    provision_title: >
      Test sets, metrics, and details about the tools used during test,
      evaluation, validation, and verification (TEVV) are documented.
    evidence_shape: >
      Policy names eval sets (mod-106 replay bundle references) per gate;
      pins metrics (rubric scores, threshold provenance); references tools
      (mod-104 rubric runner, mod-106 CI, mod-107 online loop) by build-vs-buy
      matrix row.
    residual_gap: none
    signed_by: [eval_program_owner, governance_peer]
    last_review: 2026-09-01
    
  - artefact: release_gate_policy
    artefact_ref: programs/gates/customer_facing_support_agents/policy.yaml
    framework: nist_ai_rmf
    provision: MEASURE 2.7
    provision_title: >
      AI system security and resilience are evaluated and documented.
    evidence_shape: >
      Policy names offline gate's mod-108 safety report requirement; canary/
      ramp gates' online safety metric bindings; chapter 05 investment loop
      captures security-relevant incident regression fixtures.
    residual_gap: none
    signed_by: [eval_program_owner, ai_infra_security, governance_peer]
    last_review: 2026-09-01
    
  # NIST AI RMF Manage function
  - artefact: investment_ledger
    artefact_ref: programs/investment_ledger.yaml
    framework: nist_ai_rmf
    provision: MANAGE 4.1
    provision_title: >
      Post-deployment AI system monitoring plans are implemented, including
      mechanisms for capturing and evaluating input from users and other
      relevant AI actors, appeal and override, decommissioning, incident
      response, recovery, and change management.
    evidence_shape: >
      Ledger row per P0/P1 incident with three-artefact status (fixture,
      alert, runbook diff); monthly review ceremony documented; retrospective
      enqueue process documented in chapter 05.
    residual_gap: >
      Appeal and override mechanism is currently product-owned; escalate
      to governance peer for policy-level assurance evidence.
    signed_by: [eval_program_owner, governance_peer]
    last_review: 2026-09-01
    
  # EU AI Act
  - artefact: release_gate_policy
    artefact_ref: programs/gates/customer_facing_support_agents/policy.yaml
    framework: eu_ai_act
    provision: Article 15
    provision_title: >
      Accuracy, robustness and cybersecurity.
    evidence_shape: >
      Threshold provenance clause requires accuracy thresholds be
      product- or regulator-derived (not invented); robustness thresholds
      per cohort (from cohort-preservation contract, mod-111 ch. 03);
      cybersecurity from mod-108 safety report and delegation contract
      with ai-infra-security.
    residual_gap: >
      Article 15's "level of accuracy and relevant accuracy metrics" declaration
      requires a per-surface metrics-declaration signed by governance peer;
      escalation open with governance peer 2026-08-30.
    signed_by: [eval_program_owner, governance_peer_pending]
    last_review: 2026-09-01
    
  - artefact: investment_ledger
    artefact_ref: programs/investment_ledger.yaml
    framework: eu_ai_act
    provision: Article 72
    provision_title: >
      Post-market monitoring by providers and post-market monitoring plan
      for high-risk AI systems.
    evidence_shape: >
      Investment ledger IS the post-market monitoring artefact for
      eval-relevant incidents; monthly review ceremony is the monitoring
      cadence; ledger row structure feeds Article 73 serious-incident
      reporting where applicable.
    residual_gap: >
      Article 72's requirement of a written monitoring plan is authored by
      governance peer; the ledger is referenced from the plan.
    signed_by: [eval_program_owner, governance_peer]
    last_review: 2026-09-01
    
  # ISO/IEC 42001
  - artefact: release_gate_policy + investment_ledger
    artefact_ref: >
      programs/gates/customer_facing_support_agents/policy.yaml,
      programs/investment_ledger.yaml
    framework: iso_iec_42001
    provision: Clause 9.1
    provision_title: >
      Monitoring, measurement, analysis and evaluation.
    evidence_shape: >
      Release-gate policy defines what is measured (per-gate thresholds); the
      four-gate topology defines when (offline, canary, ramp, post-ship);
      investment ledger defines the analysis-and-improvement cycle (monthly
      review); mod-110 platform slice defines the data plane.
    residual_gap: none
    signed_by: [eval_program_owner, governance_peer]
    last_review: 2026-09-01
    
  # Card slice as evidence
  - artefact: card_slice
    artefact_ref: programs/card_slices/customer_facing_support_agents/*.md
    framework: nist_ai_rmf
    provision: MEASURE 3.2
    provision_title: >
      Risk tracking approaches are considered for settings where AI risks
      are difficult to assess using currently available measurement techniques.
    evidence_shape: >
      Card slice §5 (known limitations and mitigations) captures the
      residual-risk-with-mitigation per known failure mode; card slice §7
      (recent incident summary) demonstrates the tracking of realised risks.
    residual_gap: >
      Risks not captured in the ledger (novel harms below the P1 threshold)
      require governance-peer risk register cross-reference; escalation open.
    signed_by: [eval_program_owner, governance_peer]
    last_review: 2026-09-01
```

Five things the row does:

- **Names the program artefact.** Not "the eval work"; a specific chapter-authored artefact with a file reference.
- **Names the framework provision.** Not "the NIST framework"; a specific action id or article number with a provision title.
- **Names the evidence shape.** How the artefact provides the evidence — the specific sections, references, or contents. The reader can go from the row to the artefact and back to the framework language without an intermediary.
- **Names the residual gap.** If the artefact does not fully evidence the provision, what is missing and who owns the follow-up. Missing gap analysis is worse than an incomplete artefact — it says "no gap" when there is one.
- **Names the signers and last review.** Joint authorship (eval program + governance peer) with a review timestamp. The mapping is a living document; the timestamp is what proves it.

### Which provisions the eval program's artefacts touch — a headline map

The full mapping is per-org (and per-family); the headline shape is stable enough to name. Below is the shape a mature program's mapping will have; specific ids may shift as the frameworks update.

**NIST AI RMF — Measure function.** The Measure function's actions map most directly to the release-gate policy (chapter 02), the mod-111 trade-off report evidence, and the mod-108 safety report evidence.

- **MEASURE 1.x** (Appropriate methods and metrics identified and applied) — release-gate policy names methods; chapter 03's delegation contracts name peer-provided methods.
- **MEASURE 2.x** (Systems are evaluated for trustworthy characteristics — validity, reliability, safety, security, resilience, accountability, transparency, fairness, privacy) — release-gate policy names the trustworthy characteristics per gate; mod-108 covers security / resilience; mod-107 covers reliability post-ship.
- **MEASURE 3.x** (Mechanisms for tracking identified AI risks) — card slice §5 (limitations); investment ledger.
- **MEASURE 4.x** (Feedback about efficacy of measurement is gathered and assessed) — the quarterly gate refresh (chapter 02) and the monthly investment-ledger review (chapter 05) are the feedback mechanism.

**NIST AI RMF — Manage function.** The Manage function's actions map to the investment ledger, the rollback triggers, and the delegation contracts.

- **MANAGE 1.x** (AI risks based on assessments and other analytical output are prioritised, responded to, and managed) — investment ledger prioritisation; rollback triggers from chapter 02.
- **MANAGE 2.x** (Strategies to maximise AI benefits and minimise negative impacts are planned, prepared, implemented, documented, and informed by input) — release-gate policy is the strategy document; the ledger is the implementation-and-documentation.
- **MANAGE 3.x** (AI risks and benefits from third-party entities are managed) — delegation contract with `ai-infra-mlops` on vendor risk; build-vs-buy matrix's operational-maturity axis; card slice §3 (model capability profile) on vendor model risk.
- **MANAGE 4.x** (Risk treatments, including response and recovery, and communication plans for the identified and measured AI risks are documented and monitored regularly) — investment ledger + runbook diffs; card slice §5 and §7.

**NIST AI RMF Generative AI Profile.** Adds GenAI-specific actions on top of the base functions — action ids like `MP-4.1-005` for GenAI-specific mapping actions, `MS-2.6-004` for safety-specific measure actions, `MG-4.1-002` for GenAI-specific manage actions. The mapping references the specific action ids; the peer maintains the current list.

**EU AI Act — high-risk system obligations.** For surfaces classified as high-risk under Annex III (a governance-peer determination):

- **Article 9 (Risk management system)** — release-gate policy + investment ledger are the eval-side inputs; the peer authors the risk-management-system document.
- **Article 10 (Data and data governance)** — evidence-set governance from mod-106 and mod-110; peer authors the data-governance document.
- **Article 12 (Record-keeping)** — mod-110 platform slice's trace and lineage retention; investment ledger.
- **Article 13 (Transparency and provision of information to deployers)** — card slice; peer folds into the deployer-facing documentation.
- **Article 14 (Human oversight)** — mod-109 human review workflows; peer authors the human-oversight document.
- **Article 15 (Accuracy, robustness and cybersecurity)** — release-gate policy thresholds; mod-108 safety report; delegation contract with `ai-infra-security`.
- **Article 61 / 72 (Post-market monitoring)** — investment ledger; mod-107 online loop; the monthly review ceremony.
- **Article 73 (Serious-incident reporting)** — investment-ledger rows escalate to the peer's incident-reporting pipeline when severity thresholds are met.

**EU AI Act — GPAI provider obligations.** For orgs providing general-purpose AI models on the EU market (Articles 51 – 55 and their annexes):

- **Article 53 (Obligations for providers of GPAI models)** — technical documentation, evaluation strategy, model-family capability disclosure. Model-eval-eng peer owns the model-altitude portion; app-side eval program contributes app-surface deployment context via the card slice.
- **Article 55 (Systemic-risk GPAI obligations)** — dangerous-capability evaluation, adversarial testing, cybersecurity. Model-eval-eng peer + risk-eng peer own primary; eval program provides app-surface exposure data.

**ISO/IEC 42001 — AI management system.** The standard's clauses:

- **Clause 6 (Planning)** — release-gate policy is the operational planning artefact.
- **Clause 7 (Support)** — build-vs-buy matrix + delegation contracts are the resource / competence / communication artefacts.
- **Clause 8 (Operation)** — release-gate policy execution + release runbooks + incident response via investment ledger.
- **Clause 9 (Performance evaluation)** — monthly / quarterly review cadences; investment ledger; card slice diff review.
- **Clause 10 (Improvement)** — investment ledger's continuous improvement; quarterly gate policy refresh; annual build-vs-buy matrix refresh.
- **Annex A controls** (the specific AI-relevant controls) — mapped per-control by the peer; the eval program provides the evidence where applicable.

The full mapping is longer than any single chapter can list; the exercise (exercise-05) has the reader author the per-family full mapping.

### The mapping's cadence

The mapping is a **living document** with the same discipline as the other program artefacts.

- **Monthly.** Spot-check — did any new artefact land this month that needs mapping? (A new gate policy for a new surface; a new delegation contract; a new incident ledger row that surfaces a novel provision.)
- **Quarterly.** Full walk with the governance peer — every row reviewed against the current framework language; residual gaps re-assessed; provisions re-mapped if the framework text has changed.
- **Framework refresh.** Whenever a framework publishes a revision (NIST Playbook update; EU AI Act delegated act or harmonised standard; ISO 42001 amendment), the mapping is refreshed within an agreed window (typically 30 days for major refreshes) and the peer notified.

Chapter 01's overall rhythm names the ceremonies; the mapping is the artefact several of them touch.

### The escalation shape when the mapping surfaces a gap

The mapping's most valuable output is the **residual gap** column. When a row has a non-empty gap, an escalation is either open (owned by a named peer) or the row is not signed. Three shapes:

- **The gap is closable by an eval-program artefact.** The eval program updates the artefact (adds a section to the card slice; adds an alert to the ledger; adds a threshold to the gate policy). The next mapping review closes the row.
- **The gap is closable by a peer artefact.** The eval program opens an escalation via the delegation contract (chapter 03). The peer's artefact update closes the gap; the next mapping review closes the row.
- **The gap is a policy interpretation question.** The eval program cannot close; the governance peer authors an interpretation or the peer escalates further (to internal legal, to external counsel, to the regulator's guidance channel). The row stays open with `residual_gap: interpretation pending` until resolved.

The escalation shapes are documented in chapter 03's contracts; the mapping row references them by contract-side artefact ids.

### The mapping as a first-class audit artefact

When an audit walkthrough happens — internal risk review, third-party procurement security review, regulator submission — the mapping is the artefact walked through, not the individual dashboards. The audit shape:

- The auditor picks a framework provision (typically at random from a sample the auditor has pre-chosen).
- The governance peer (with the eval program owner on hand) opens the mapping row for the provision.
- The row names the artefact and the evidence shape.
- The auditor is walked through the artefact — the release-gate policy file; the investment ledger entries; the card slice section.
- The auditor asks follow-up questions; the artefact answers them (or an escalation is opened if the artefact does not).

The mapping is what makes the walkthrough tractable. Without it, the walkthrough is an evidence-hunt; with it, the walkthrough is a *reading*.

### The framework-refresh-tracking process

Frameworks evolve. Three sources the program tracks:

- **NIST AI RMF Playbook.** Published on the NIST Trustworthy AI Resource Center; new actions and revised action language publish periodically.
- **EU AI Act.** The Regulation itself is stable; the delegated acts, implementing acts, and Commission guidance (harmonised standards, Code of Practice) are the live surface. The AI Office publishes these.
- **ISO/IEC 42001 and adjacent standards.** ISO/IEC 42001 is the base; ISO/IEC 23894 (risk management guidance) and ISO/IEC 5259 (data quality) are companions. Amendments publish periodically.

The governance peer owns the primary tracking (chapter 03's contract); the eval program subscribes to updates via the peer's channel. Whenever the peer signals a refresh, the mapping's next quarterly review incorporates.

### Anti-patterns to avoid

**"Mapping in a spreadsheet nobody signs."** The mapping exists but no signer, no review cadence, no timestamps. First audit finds it stale. Fix: living document with signers and cadence.

**"Mapping by artefact only, not by provision."** Each artefact is listed with the provisions it touches, but no reverse index. When the auditor asks "what evidences MEASURE 2.7?" the program cannot answer without walking every artefact. Fix: the mapping is a per-provision index (one row per artefact-to-provision link).

**"Mapping without gaps."** Every row claims "no gap"; the gap column is empty everywhere. Either the program is perfect (unlikely) or the gap analysis was skipped (likely). Fix: honest gap analysis; escalations owned.

**"Eval program authors the mapping alone."** The governance peer is out of the loop; the peer's interpretation of the framework is not incorporated; the mapping drifts from the peer's actual audit posture. Fix: joint authorship; peer as co-signer.

**"Mapping refreshed only at audit time."** The mapping goes stale between audits; the audit-time refresh is a scramble. Fix: quarterly refresh cadence; framework-refresh triggers.

**"Provision interpretation without escalation."** A provision is unclear; the eval program interprets it internally; the interpretation is wrong at audit time. Fix: the eval program does not interpret framework language; escalate to the peer.

### The governance-peer relationship, revisited

Chapter 03 covered the delegation contracts with the governance peers. The framework mapping is where the contracts pay off — every mapping row's signer set includes the governance peer; every residual gap is either an eval-program action or a peer action; every framework refresh triggers a joint review.

Two things worth naming again:

- **Governance is not eval.** The governance peer authors policy, interprets regulation, faces the regulator, and signs the org-wide card. The eval program produces evidence in shapes the peer consumes. Chapter 07's mapping is the shape of that consumption.
- **The eval program's framework mapping is not the org's compliance stance.** The org's compliance stance is broader — vendor contracts, employee training, third-party audits, HR policies, physical-security controls. The eval program's mapping covers the eval-adjacent provisions; the peer covers the rest.

The mapping's job is to make the eval-adjacent portion of the org's compliance stance *observable*, *defensible*, and *refreshable*. Everything downstream — audit walkthroughs, customer procurement reviews, regulator submissions — reads from the mapping.

### The end-of-chapter posture

By the end of this chapter you can:

- Author a **framework mapping** for one app surface family — one row per (program artefact, framework provision) link, with evidence shape, residual gap, signers, and last-review timestamp.
- Bind the mapping to the **program's artefacts** (release-gate policy, delegation contracts, build-vs-buy matrix, investment ledger, card slice) so every artefact has a framework footprint.
- Run the **quarterly mapping review** with the governance peer — walk every row against current framework language; refresh gaps; incorporate framework refreshes.
- Handle an **audit walkthrough** with the mapping as the primary artefact — the auditor picks a provision; the mapping opens the row; the row names the artefact; the artefact answers.
- **Escalate gaps** cleanly — closable-by-eval, closable-by-peer, policy-interpretation-pending; each has an owner and a follow-up shape.

## Summary

- The eval program does not own **framework compliance** (owned by the governance peer). The eval program owns the **framework mapping** — the crosswalk between program artefacts (chapters 02 – 06) and the specific framework provisions the artefacts evidence.
- Three frameworks in scope: **NIST AI RMF** (AI 100-1) + Generative AI Profile (AI 600-1) — Measure and Manage functions especially; **EU AI Act** (Reg. (EU) 2024/1689) — high-risk system obligations (Articles 9 – 15, 61 – 73) and GPAI provider obligations (Articles 51 – 55); **ISO/IEC 42001:2023** — AI management system clauses 6 – 10 and Annex A controls.
- **Mapping shape**: one row per (artefact, provision) pair, with artefact reference, provision title, evidence shape, residual gap, signers, last review. The mapping is a living document.
- Program artefacts map primarily to: **release-gate policy** → NIST Measure 1/2, EU Article 15, ISO Clause 8; **investment ledger** → NIST Manage 4, EU Articles 61/72/73, ISO Clauses 9/10; **card slice** → NIST Measure 3, EU Article 13, ISO Clause 7; **delegation contracts** → NIST Manage 3, EU Article 25 (deployer obligations for shared responsibility), ISO Clause 7 (competence, communication); **build-vs-buy matrix** → NIST Manage 3, ISO Clauses 7 and 8.
- **Cadence**: monthly spot-check; quarterly full walk with the governance peer; framework-refresh-driven update within an agreed window.
- **Residual-gap escalation**: closable-by-eval-program (update the artefact); closable-by-peer (delegation-contract escalation); policy-interpretation-pending (governance peer authors or escalates further).
- The mapping is the **primary audit artefact** — the walkthrough reads from it, not from individual dashboards.
- **Framework-refresh tracking** is a peer-primary responsibility; the eval program subscribes via chapter 03's governance contracts.
- **Anti-patterns**: unsigned spreadsheet, per-artefact-only index, gap-free mapping, eval-alone authorship, audit-time-only refresh, in-house provision interpretation.
- Governance is not eval; the eval program's mapping covers the eval-adjacent portion of the org's compliance stance, and the governance peer covers the rest.

The module ends here. The seven chapters together are the program-owner posture — the release-gate architecture, the delegation contracts, the build-vs-buy platform matrix, the incident-driven investment loop, the product-side card slice, and the governance-framework mapping — that turns eleven modules of practitioner work into an eval program a director defends at business review, a peer specialist contracts with, an auditor walks through, and a regulator reads.
