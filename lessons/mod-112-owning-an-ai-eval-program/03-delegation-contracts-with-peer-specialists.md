# Delegation Contracts With Peer Specialists: The Machine-Readable Boundary

## Motivation

Chapter 01 named the *sprawl* failure shape: the eval team absorbs peer scope because "we might as well have the number," and six months in the eval team is doing the peer's job with the eval team's headcount. The failure is not a moral one — it is a *contracts* failure. Where the boundary between the eval program and a peer specialist is not written down, the boundary is negotiated per-ticket, and the ticket-by-ticket negotiations always drift toward the eval team absorbing.

This chapter formalises the boundary. A **delegation contract** is a written, versioned agreement between the eval program and a peer specialist that names what each side produces for the other, on what cadence, in what format, with what escalation shape. Contracts turn ad-hoc negotiation into a reviewable artefact and stop the drift.

Six peer contracts the program owner authors and maintains — one per peer track this module interfaces with. The contracts follow a common shape (§ "Anatomy of a delegation contract") and differ in *what* each side owes the other and *when*.

Two symptoms show a peer contract is missing:

- **The peer produces work by asking, not by policy.** Every time the eval program needs the peer's evidence, it starts a fresh negotiation about scope, timeline, and format. The peer is generous or resistant depending on the week; the eval program's dependencies on the peer are un-plannable.
- **The eval program shadows peer work.** The eval team runs its own version of the peer's suite ("just to have a check"), maintains its own version of the peer's artefacts, and duplicates effort because the peer's outputs are not shaped to consume. Six months in, the eval team has a smaller worse-quality version of the peer's stack, and the peer has no incentive to fix theirs because the eval team is covering.

Both symptoms are contract failures. This chapter closes them.

## Core concepts

### The peer roster this module interfaces with

Six peers. Each has its own remit, its own artefacts, and its own escalation shape. The contracts differ per peer, but all follow the anatomy in the next section.

| # | Peer role | Family / level | What the peer owns | What the eval program owes back |
|---|---|---|---|---|
| 1 | `model-evaluation-engineer` | Model-Development, L30 | Model-altitude eval — offline benchmark suites (MMLU, GPQA, HumanEval, MMMU, LiveCodeBench), judge-model training, dangerous-capability evals, MLPerf serving benchmarks (see mod-111 ch. 07) | App-altitude cohort regressions, judge-overlap diagnostics, capability-profile-request tickets scoped to real shipping decisions |
| 2 | `ai-risk-engineer` | Governance, L30 | Harm model, adversary personas, red-team data generation, dangerous-capability escalation | App-surface findings that surface new harm categories, product-side impact analyses, exposure-attribution when a harm shows up in prod |
| 3 | `ai-evaluation-engineer` (peer, Governance) | Governance, L25 | Release-assurance framework authoring, regulator interface, org-wide model/system card | Per-surface card slices (chapter 06), per-family gate outcomes, framework mappings (chapter 07) |
| 4 | `ai-governance-analyst` | Governance | Org-wide AI policy, third-party attestations, vendor risk assessments, enforcement escalation | Evidence bundles per policy-adjacent question, incident ledger entries with regulatory implications |
| 5 | `ai-infra-security` | Infrastructure, L30 | Adversarial-eval infrastructure, judge / evaluator supply-chain, secure trace storage, key management for the eval pipeline | Attack findings from the mod-108 safety report, judge-supply-chain SBOM requirements, trace-access audit expectations |
| 6 | `ai-infra-mlops` / `ai-infra-ml-platform` | Infrastructure | CI / CD integration surface, deployment machinery the release gates bind to, the trace-collection substrate | Gate policy bindings, mod-106 replay-bundle CI hooks, mod-107 online-loop infra requirements, mod-110 platform-slice storage shape |

Two things to notice:

- **The peer roles are not "sub-teams of eval."** They are distinct roles with distinct owners. The eval program is one of *their* consumers, and they are consumers of *the eval program*. Reciprocity is symmetric.
- **The contracts are per-peer, not per-request.** The relationship is standing; the request-shape is derived. If every request re-negotiates the contract, the contract is not doing its job.

### Anatomy of a delegation contract

Every peer contract has the same eight sections. Consistency across contracts matters for the same reason consistency across gate policies matters: the reader can walk any contract without relearning the shape.

```yaml
peer: model-evaluation-engineer
version: 2026-Q3
signed:
  - eval_program_owner: alex
  - peer_owner: casey
  - date: 2026-07-14
review_cadence: quarterly
next_review: 2026-10-01

scope:
  # What this contract covers and what it does not.
  covers:
    - Model-altitude capability profile requests scoped to shipping decisions on {app_families}
    - Judge-overlap diagnostics on rubrics used by {app_families}
    - Model-swap regression coverage on {app_families}
  does_not_cover:
    - Peer's dangerous-capability eval program (owned end-to-end by peer)
    - App-altitude cohort authoring (owned end-to-end by eval program)
    - Vendor procurement decisions (owned by peer with input from eval program)

what_the_peer_produces:
  - artefact: capability_profile_snapshot
    shape: peer's standard schema; JSON payload per model_snapshot
    cadence: monthly refresh; on-demand for a shipping decision within 5 business days
    lands_in: mod-110 store, key `capability_profiles.model_snapshot`
    consumed_by:
      - chapter 02 gate: offline gate for model_snapshot and model_family change classes
      - chapter 06 card slice: "model capability profile" section
  - artefact: judge_overlap_diagnostic
    shape: paired-comparison rubric outputs on peer's held-out set
    cadence: quarterly; on-demand for a rubric-methodology question
    lands_in: mod-110 store, key `judge_diagnostics.rubric_id`
    consumed_by:
      - mod-104 chapter 05 (judge calibration)
      - chapter 06 card slice: "judge methodology" section

what_the_eval_program_produces:
  - artefact: cohort_regression_report
    shape: mod-111 chapter 04 delta report on peer's benchmark cohorts
    cadence: on-demand when app-altitude finds a regression attributable to model altitude
    lands_in: peer's tracker; referenced from mod-110 `escalations` table
  - artefact: judge_overlap_result
    shape: eval program's paired-comparison result against peer's diagnostic
    cadence: on receipt of peer's diagnostic
    lands_in: mod-110 store; peer's tracker

interface_channels:
  - kind: escalation_ticket
    template: templates/escalation-to-model-eval-eng.md
    tracker: jira://EVAL
    sla_ack: 2 business days
    sla_scoping: 5 business days
  - kind: joint_review
    cadence: monthly
    duration: 30 min
    agenda: outstanding escalations + upcoming shipping decisions + contract drift

escalation:
  # What to do when the contract is not being met on either side.
  eval_program_owes_peer_late:
    trigger: any owed artefact > SLA by > 3 business days
    action: escalate to eval program owner; add to next quarterly review
  peer_owes_eval_program_late:
    trigger: any owed artefact > SLA by > 3 business days
    action: escalate to peer owner and eval program owner's manager; add to next quarterly review; if repeated, escalate to org's cross-team engineering leadership

bootstrap_state:
  # Whether the peer function is fully in place; the plan if not.
  peer_exists: yes
  peer_bootstrapping: no
  eval_program_bootstrapping_peer: no
  # If any is "yes," a hand-off timeline lives in the linked bootstrap plan.
  bootstrap_plan: n/a
```

Eight sections. Each is load-bearing.

- **Header.** Peer, version, signers, review cadence. The contract is versioned; the review cadence is what keeps it alive.
- **Scope.** What the contract covers and what it does not. Naming the exclusions is as important as naming the inclusions — most drift happens in the "well, this feels related to your work" grey zone.
- **What the peer produces.** Named artefacts, shapes, cadences, storage locations, consumers. The consumer list is what makes the contract auditable: if a downstream chapter (release gate, card slice, framework mapping) says it depends on this artefact, the contract confirms it is being produced.
- **What the eval program produces.** Symmetric. Every peer contract has *this side* named; a contract that is one-way is a contract that will erode.
- **Interface channels.** How work is initiated between the two sides. Escalation-ticket templates, joint-review cadence, SLAs. Not "we DM each other on Slack."
- **Escalation.** What happens when either side misses. Missing this section is how a contract stops being enforceable.
- **Bootstrap state.** Whether the peer function is fully staffed; the plan if it is being stood up. This is the single most important section for a program in a young org where the peer roles are not all in place — the contract acknowledges the reality and names the hand-off plan.

### Contract 1 — `model-evaluation-engineer` (Model-Development, L30)

The model-eval-eng peer is the model-altitude counterpart to the eval program. Mod-111 chapter 07 walked the practitioner-level relationship (capability-profile hand-offs, replacement regression, MLPerf reconciliation); the program-level contract makes it a standing agreement.

**What the peer produces for the eval program:**
- **Capability profile per candidate model.** For every model snapshot the eval program is about to gate a swap-in on, the peer produces the model-altitude capability profile (relevant benchmark scores against the peer's suite; the peer's judge-model performance if the model is being used as a judge). Cadence: monthly refresh; on-demand within an agreed SLA for a shipping decision.
- **Judge-overlap diagnostic per rubric.** For every rubric the eval program uses that overlaps with the peer's held-out set (usually the cross-cutting ones — helpfulness, factuality, harmlessness), the peer produces a paired-comparison diagnostic so the eval program can detect judge drift.
- **Model-swap regression coverage plan.** For every model family transition in scope (frontier-vendor A → frontier-vendor B; OSS → hosted; etc.), the peer produces the coverage plan — which model-altitude benchmarks confirm the swap is plausible before the eval program's app-altitude replacement regression runs.

**What the eval program produces for the peer:**
- **Cohort regression report.** When the app-altitude eval finds a regression attributable to model altitude (mod-111 chapter 07's escalation shape), the eval program produces the paired eval set, the per-cohort deltas, and the lineage-key set the peer needs to reproduce.
- **Judge-overlap result.** When the peer's diagnostic lands, the eval program runs its own rubric on the diagnostic set and returns the paired result so the peer can calibrate.
- **Real-world usage signal.** The eval program produces (monthly) a summary of which model snapshots are in production, on which surfaces, at what traffic share, and which are being deprecated. The peer's benchmark refresh cadence follows real usage rather than "which model is in the news."

**The boundary the contract enforces:** The eval program does not run MMLU sweeps. The peer does not author app-altitude cohort rosters or sign shipping decisions. Judge-model training is peer's; judge-model *use in a rubric* is the eval program's.

### Contract 2 — `ai-risk-engineer` (Governance, L30)

The risk-eng peer owns the harm model, adversary personas, and red-team data generation. Mod-108 chapter 01 walked the practitioner-level delegation; the program-level contract makes it standing.

**What the peer produces for the eval program:**
- **Harm-model refresh.** Quarterly, the peer refreshes the harm-model taxonomy — the categories of harm the safety report addresses. Categories may split, merge, or move as the risk landscape evolves.
- **Adversary persona library.** The set of adversary shapes the red-team data targets — script-kiddie, social-engineer, motivated-technical-attacker, insider, etc. The mod-108 safety report's attack shapes read from this.
- **Red-team data corpus.** The corpus of prompts and scenarios the offline safety gate scores against. Refreshed on the peer's cadence with a documented change-log.
- **Escalation on new dangerous-capability signals.** When the peer's dangerous-capability watch surfaces a new signal (e.g., a new category of biosecurity uplift the peer is now tracking), the eval program receives it within an agreed SLA so the release gates can incorporate.

**What the eval program produces for the peer:**
- **Product-surface impact analysis.** When the peer's new harm category lands, the eval program produces the per-surface impact analysis — which surfaces have exposure, what the mod-108 findings show, what the release-gate mapping needs to add.
- **Exposure attribution.** When a harm shows up in production (via mod-107's online safety findings or via a real incident), the eval program produces the exposure attribution — which surface, which change class, which lineage keys — so the peer can trace back to methodology gaps.
- **Real-world attack-signal feedback.** The eval program produces (monthly) a summary of the guardrail-trigger rate per attack category from mod-107; the peer's next red-team refresh weights the corpus toward the categories real attackers are attempting.

**The boundary the contract enforces:** The eval program does not author the harm-model taxonomy or the adversary personas. The peer does not run app-surface guardrails or sign per-surface release gates. Corpus authorship is peer's; corpus *use in a gate* is the eval program's.

### Contract 3 — `ai-evaluation-engineer` (peer, Governance, L25) and Contract 4 — `ai-governance-analyst`

The governance peers are the release-assurance and regulator-interface side of the triangle. Two roles; often the contracts are authored together because the ownership boundary between them (release assurance vs. broader governance policy) varies per org. The distinction:

- The `ai-evaluation-engineer` peer at L25 owns the **release-assurance framework** — the set of assurance activities per surface that ties eval evidence to the release decision. They are the *first* consumer of the eval program's gate outcomes and card slices.
- The `ai-governance-analyst` owns the **org-wide AI policy**, third-party attestations, vendor risk assessments, and the enforcement escalation path when a shipped surface later trips a policy line. They are the *second* consumer.

The two contracts share most of their shape; the differences are in the artefact list.

**What the peers produce for the eval program:**
- **Regulator-inherited requirements.** For surfaces in regulated scope (EU AI Act high-risk, US financial regulation, medical, etc.), the governance peers produce the specific accuracy / robustness / cybersecurity thresholds the release gates must meet. Chapter 02's regulator-derived thresholds come from here.
- **Card template.** The org-wide model / system card template (per surface family) that the eval program's card slice folds into. Card templates evolve; the peers publish new versions with change-logs.
- **Framework mapping crosswalks.** The current mapping between the org's artefacts and the frameworks (NIST AI RMF, EU AI Act, ISO/IEC 42001). Chapter 07's mapping reads from this.
- **Policy interpretation.** For grey-zone questions ("does this new surface count as an in-scope surface under our AI policy?"), the peers produce the interpretation. The eval program does not make policy calls.

**What the eval program produces for the peers:**
- **Per-surface card slices.** Chapter 06's deliverable. The eval evidence in a shape the governance peer folds into the org-wide card with zero rewrites.
- **Release-gate outcome ledger.** Every gate outcome per surface — offline, canary, ramp, post-ship — indexed for the peers' assurance evidence.
- **Incident-adjacent evidence.** For any incident with governance implications (a real regression on a regulated surface; a safety event needing regulator notification under EU AI Act Article 73), the eval program produces the evidence bundle within an agreed SLA.
- **Framework mapping evidence.** Chapter 07's artefacts, per surface, in the schema the governance peers consume for external attestations.

**The boundary the contract enforces:** The eval program does not author policy or interpret regulation. The governance peers do not author release-gate thresholds independently of the eval program (they *inherit* to product-committed and safety-mandated thresholds; the eval program pins the numbers).

### Contract 5 — `ai-infra-security` (Infrastructure, L30)

The infra-security peer owns the security posture of the eval pipeline itself. Two distinct concerns: **adversarial-eval infrastructure** (the infrastructure that runs the mod-108 safety suite) and **judge / evaluator supply-chain** (the trust model for the judge models and evaluator scripts the pipeline uses).

**What the peer produces for the eval program:**
- **Adversarial-eval isolation.** The infrastructure that runs prompt-injection payloads, jailbreak corpora, and canary tokens without leaking payloads into production, without cross-contaminating tenant data, and without opening the eval pipeline itself to attack.
- **Judge supply-chain policy.** The trust model for judge models — where they come from (vendor-hosted, self-hosted, third-party), what SBOM they carry, how updates flow, how model-swap regressions are gated. Chapter 04 of mod-104 covered the practitioner side; the program-level policy comes from here.
- **Trace-storage security controls.** Access controls, key management, retention policy, and audit logging for the mod-110 platform slice's trace storage. Especially load-bearing when traces contain user data.
- **Attack-signal enrichment.** For findings from the mod-108 safety report, the peer produces the mapping to MITRE ATLAS techniques and (where relevant) real-world attack observations from the org's SIEM.

**What the eval program produces for the peer:**
- **mod-108 findings feed.** The stream of safety findings from the offline and online gates, in the schema the peer's SIEM consumes.
- **Judge-supply-chain requirements list.** The set of judge / evaluator components the eval pipeline depends on, per surface, refreshed monthly. Feeds the peer's SBOM tracking.
- **Trace-access audit expectations.** The eval program's access-pattern to the mod-110 platform slice — who reads what, how often — so the peer can size the audit logging and detect anomalies.

**The boundary the contract enforces:** The eval program does not author the adversarial-eval infrastructure or maintain the judge supply-chain policy. The peer does not author rubrics or interpret safety findings for product impact. Infrastructure is peer's; *use* of the infrastructure to run a gate is the eval program's.

### Contract 6 — `ai-infra-mlops` / `ai-infra-ml-platform` (Infrastructure)

The MLOps and ML-platform peers own the CI / CD substrate and the ML platform layer the eval pipeline runs on. Often two roles or one role with two hats; the contract shape is the same.

**What the peers produce for the eval program:**
- **CI integration surface.** The CI machinery hooks the mod-106 replay-bundle gate binds to. The peer maintains the runners, the pipeline shape, the artefact-storage integration.
- **Canary / ramp machinery.** The traffic-shaping and flag-flipping infrastructure the release gates from chapter 02 bind to. The peer maintains the flag service, the ramp-scheduler, the rollback machinery.
- **Trace-collection substrate.** The OTel-GenAI collector, the platform slice's ingestion pipeline (mod-110), the platform slice's query surface. The peer owns the data plane; the eval program owns the schema and the read-shape.
- **Deployment-lineage service.** Every surface's deployment history — what was live, when, with what lineage keys. The mod-107 post-ship gate reads this; chapter 05's investment ledger reads this.

**What the eval program produces for the peers:**
- **Gate policy binding requirements.** Chapter 02's policies in a shape the CI / canary / ramp machinery can enforce (typically a machine-readable YAML consumed by the peer's pipeline definition language).
- **Storage-shape requirements.** The mod-110 schema (from that module's chapter 02) in a shape the peer's platform team can implement / operate.
- **SLO expectations.** The gate's response-time expectations (offline gate < X min; canary evaluation window measured in hours; rollback trigger latency < Y minutes) so the peers can size the infrastructure.

**The boundary the contract enforces:** The eval program does not author the CI system, the flag service, or the collector. The peers do not author gate policies or interpret gate outcomes. Infrastructure is peer's; *policy* over the infrastructure is the eval program's.

### The bootstrap-state clause — the young-org reality

Every contract has a **bootstrap state** section. In a mature org, all six peer functions exist and are staffed; the bootstrap flags are all "no" and the plans are "n/a." In a young org — which is where most of these programs are built — some peers are being stood up, some are absent, and the eval program is temporarily doing the peer's work.

Two acknowledgements the contract makes:

- **The peer function is being stood up.** The contract names the timeline (typically two to four quarters), the hiring plan, and the artefacts the eval program is *temporarily* producing. The eval program's absorption is time-boxed; the hand-off date is in the contract.
- **The eval program is bootstrapping the peer's artefacts.** For an artefact the peer will eventually own but the eval program produces today, the contract names the artefact, the acceptance criteria for the peer to take it over, and the hand-off date.

Bootstrap is not the same as absorption. Bootstrap has a hand-off date; absorption has none. A bootstrap plan that has slipped its hand-off date twice in a row is a bootstrap plan that has silently become absorption — the quarterly review flags it and either extends explicitly (with new evidence for the timeline) or triggers an escalation to the org's engineering leadership to unstick the hiring.

The contract's honesty about bootstrap is what keeps the program's scope crisp. Every artefact the eval program produces has an owner; when the owner is the eval program today because the peer function does not exist yet, the contract says so and the escape hatch is planned.

### Contract review — the quarterly rhythm

Every contract has a review cadence in the header. Quarterly is the default. The review is a joint session between the eval program owner and the peer owner; it takes 30 – 45 minutes and produces a diff.

Review agenda:

- **Artefact production and consumption.** Did every artefact land within its SLA? Are there consumers that never actually read the artefact? Are there consumers not on the list who should be?
- **Interface channels.** Are the ticket templates still fit-for-purpose? Are the joint reviews producing useful decisions?
- **Escalation events.** Did any escalation trigger fire this quarter? Was it resolved appropriately? Does the trigger threshold need adjustment?
- **Bootstrap state.** Are the bootstrap plans on track? Have any absorbed-artefacts drifted from their hand-off timeline?
- **Contract drift.** Has either side started producing something not in the contract, or stopped producing something in the contract? Is the drift correct (update the contract) or wrong (fix the drift)?

The review produces a versioned diff — a new `version:` in the header, an updated `next_review:` date, and a summary of changes filed alongside `history/`. The contract is a living artefact.

### Anti-patterns to avoid

**"Verbal delegation."** The eval program owner and the peer owner agree over lunch that the peer will produce X. Six months later either side leaves; the agreement leaves with them. Fix: written, versioned contract; signers on both sides.

**"Contract without escalation."** The contract names the artefacts but not what happens when either side misses. First miss is tolerated; second miss becomes the norm; the contract erodes. Fix: escalation section named; triggers explicit.

**"Bootstrap without hand-off date."** The eval program is temporarily producing an artefact; six months in, "temporarily" has no defined end. Fix: every bootstrap plan has a hand-off date, and slippage triggers an explicit renewal or escalation.

**"Consumer-less artefacts."** The contract names an artefact the peer produces monthly; no downstream chapter names it as a dependency. The peer is doing work no one reads. Fix: every artefact names its consumers; consumer-less artefacts are pruned at the quarterly review.

**"One-way contracts."** The peer produces for the eval program; the eval program produces nothing in return. The relationship becomes extractive; the peer under-invests. Fix: symmetric contracts; both sides produce for the other.

**"Contract sprawl."** Every ticket produces a mini-contract; the eval program is maintaining 200 tiny agreements. Fix: one contract per peer role; ticket-shape derived from the contract, not authored fresh.

### The end-of-chapter posture

By the end of this chapter you can:

- Author a **delegation contract** with each of the six peer specialists using the anatomy the chapter walks — header, scope, what the peer produces, what the eval program produces, interface channels, escalation, bootstrap state.
- Bind each contract's artefact list to the **release-gate architecture** (chapter 02), the **investment ledger** (chapter 05), the **card slice** (chapter 06), and the **framework mapping** (chapter 07) so every artefact has a named consumer.
- Run a **quarterly review** with each peer — walk the agenda, produce a versioned diff, and refresh the contract.
- Distinguish **bootstrap** (time-boxed, hand-off date) from **absorption** (permanent, scope-creep) and act on drift before it becomes structural.

Chapter 04 opens the build-vs-buy platform matrix — the decision-shape that decides which pieces of the eval-platform stack are bought vs. built, refreshed on its own cadence.

## Summary

- A **delegation contract** is a written, versioned agreement between the eval program and a peer specialist. Contracts turn ad-hoc negotiation into a reviewable artefact and stop the sprawl.
- **Six peer contracts** the program owner authors: `model-evaluation-engineer` (model altitude), `ai-risk-engineer` (harm model & red-team data), `ai-evaluation-engineer` (peer, Governance L25) and `ai-governance-analyst` (release-assurance & regulator interface), `ai-infra-security` (adversarial eval & judge supply chain), `ai-infra-mlops` / `ai-infra-ml-platform` (CI & platform integration).
- **Eight-section anatomy**: header, scope (covers / does not cover), what the peer produces, what the eval program produces, interface channels, escalation, bootstrap state, review cadence. Every contract is symmetric — both sides produce for the other.
- The **bootstrap-state clause** acknowledges the young-org reality — peer functions being stood up, artefacts temporarily absorbed by the eval program — with a hand-off date and a plan. Bootstrap is not absorption.
- **Quarterly review** produces a versioned diff — artefact production audited, escalation events reviewed, bootstrap plans checked, drift resolved.
- **Anti-patterns**: verbal delegation, contract without escalation, bootstrap without hand-off date, consumer-less artefacts, one-way contracts, contract sprawl.
- The contracts bind the release-gate architecture (ch. 02) to the peer artefacts that feed it; the investment ledger (ch. 05) to the peer artefacts that consume it; the card slice (ch. 06) and framework mapping (ch. 07) to the governance peers that consume both.

Chapter 04 opens the build-vs-buy platform matrix — the decision that pins which parts of the eval-platform stack the program buys and which it builds.
