# Why Owning an AI Evaluation Program: The Program-Owner Posture the Track Builds Toward

## Motivation

The previous eleven modules built individual surfaces: trace instrumentation (mod-102), trajectory eval (mod-103), LLM-as-judge (mod-104), RAG eval (mod-105), CI gate (mod-106), online loop (mod-107), safety report (mod-108), human review (mod-109), the platform slice (mod-110), the cost / latency / quality trade-off report (mod-111). Every one of them is a load-bearing artefact and every one of them, on its own, is not a program.

A **program** is what you defend to a director at business review. It is the release-gate architecture the whole product portfolio uses, the delegation contracts that keep six other teams working *with* you and not *around* you, the build-vs-buy decisions that decide whether your evaluation stack is an org asset or an ongoing tax, the incident-driven investment cadence that turns every production regression into permanent coverage, the product-side card slice that the governance peer signs and hands to the regulator, and the mapping between your artefacts and the frameworks (NIST AI RMF, EU AI Act, ISO/IEC 42001) the org will be audited against.

This module is not about producing another eval; it is about **owning the eval program**. The chapters walk the six load-bearing surfaces a program owner defends:

- The **release-gate architecture** (chapter 02) — the way an entire product surface family is gated by eval evidence, with explicit thresholds and rollback criteria.
- The **delegation contracts** (chapter 03) — the machine-readable contracts with peer specialists (`model-evaluation-engineer`, `ai-risk-engineer`, the governance peers, `ai-infra-security`, `ai-infra-mlops` / `ai-infra-ml-platform`) that keep the program from either absorbing peer work or leaking eval work to peers.
- The **build-vs-buy decision** (chapter 04) — how the org picks between running Weave / Phoenix / Langfuse / Braintrust / Promptfoo / DeepEval / RAGAS / Humanloop / Galileo / Patronus / Vertex AI Evaluation / Bedrock Evaluations / Azure AI Foundry Evaluations, and where the boundaries between them and the platform slice (mod-110) sit.
- The **incident-driven eval investment loop** (chapter 05) — the discipline that turns a real production incident into a permanent regression fixture, an online alert, and a runbook update, so the same failure never recurs unnoticed.
- The **product-side model / system card slice** (chapter 06) — the composed card slice the eval program authors, signed by product and eng, that the governance peer folds into the org-wide card and hands to internal review and (where applicable) regulators.
- The **program-to-framework mapping** (chapter 07) — how the program's artefacts map to NIST AI RMF Measure / Manage, EU AI Act GPAI provider obligations, and ISO/IEC 42001 controls, knowing the governance shape itself is owned by the `ai-evaluation-engineer` (Governance family, L25) peer and by `ai-governance-analyst`.

Three symptoms show a team is running eval without a program:

- **Every release argues its own gate.** Team A ships when quality is "good enough"; team B ships when the on-call rotation OKs it; team C ships whenever the vendor demo looks better. Nothing composes into an org-wide picture, and when a regression happens the retrospective cannot answer "was the release gate consistent with our stated policy?" because there is no stated policy.
- **Peer teams route work around evaluation.** The safety team runs its own red-team suite because "eval is slow." The MLOps team maintains its own regression fixtures because "eval doesn't cover the deployment path." The governance analyst writes the model card slice from scratch because "eval's reports don't map to our template." Every one of those is a delegation contract that never got authored.
- **The eval platform is a rewrite every 18 months.** The team adopted Weave because Phoenix was too heavy; six months later Weave is too limited and everything is being ported to Braintrust; six months after that Braintrust's costs cross the buy line and the team is rebuilding on Langfuse. The build-vs-buy decision was made once, informally, and re-argued in every review.

Each symptom is a **program-shape** failure disguised as a tools, staffing, or scope failure. This module builds the program shape so the disguises are impossible.

## Core concepts

### The program-owner posture

The AI Evaluation Engineer at level 30 (the terminal level of the app-side eval track) is not the person who *runs* every eval — that would not scale past two product surfaces. They are the person who owns the *program* — the release gates, the delegation contracts, the platform decisions, the incident investment loop, the card slices, and the framework mapping — such that the eval work itself is done by a mix of specialised peers, embedded eval engineers on product teams, and self-service tools the peers use.

The posture is discriminating: the level-30 owner does *less* eval implementation and *more* eval decision-shaping than a level-20 or level-25 practitioner. The output is not "I scored a rubric on these traces this week"; the output is "we shipped this quarter's decisions against the gates we published, with the delegation contracts intact, on the platform we picked with our eyes open, and with a card slice the governance peer accepted with zero rewrites."

Four things the posture holds up:

1. **A published release-gate policy** that every product surface in scope maps to — even if the thresholds differ per surface. The policy is document + code, not a wiki page. Chapter 02.
2. **A delegation contract set** with every peer the program interfaces with — machine-readable, versioned, referenced from tickets. Chapter 03.
3. **A build-vs-buy platform matrix** revisited on a documented cadence (usually annually), with the decision defensible against last cycle's evidence. Chapter 04.
4. **An incident investment loop** running on the cadence of production incidents — every P0 / P1 in scope produces a regression fixture, an alert, and a runbook diff. Chapter 05.

The card slice (chapter 06) and the framework mapping (chapter 07) are how these four surfaces publish externally.

### The plane analogy, extended upward

Mod-111 introduced the app-altitude / model-altitude / fleet-altitude split. This module extends it upward — the *program* altitude sits above the app altitude and is what the org's leadership sees.

```
   +-----------------------------------------------------+
   |   program altitude — THIS MODULE                    |
   |   (AI Evaluation Engineer, L30 program-owner)       |
   |                                                     |
   |   release-gate architecture across product surfaces |
   |   delegation contracts with peer specialists        |
   |   build-vs-buy decision on the eval platform stack  |
   |   incident-driven investment cadence                |
   |   product-side card slice → governance hand-off     |
   |   framework mapping (NIST / EU AI Act / ISO 42001)  |
   +----------------+------------------------------------+
                    |
                    | reads down; publishes back up
                    v
   +-----------------------------------------------------+
   |   app altitude — mod-101 … mod-111                  |
   |   (AI Evaluation Engineer, L20 – L30 practitioner)  |
   |                                                     |
   |   per-surface rubrics, RAG eval, gates, online loop |
   |   safety report, human review, platform slice       |
   |   cost / latency / quality trade-off report         |
   +----------------+------------------------------------+
                    |
                    | reads down; publishes back up
                    v
   +-----------------------------------------------------+
   |   model altitude — model-evaluation-engineer (L30)  |
   |   fleet altitude — ai-infra-performance             |
   +-----------------------------------------------------+
```

Program altitude reads *down* into the app-altitude artefacts (mod-101 to mod-111's outputs are the evidence base) and publishes *up* into the org's release, governance, and regulator-facing surfaces. It does not re-produce the app-altitude artefacts; it composes them.

The distinction that matters: **program altitude is the altitude at which decisions get *policy*, not just *outcomes*.** Individual shipping decisions live at app altitude. The policy that all shipping decisions must follow lives at program altitude. Miss the distinction and the program is a debating society or an authoritarian bottleneck — both fail.

### The six load-bearing surfaces

One chapter per surface. Each is required for the program to be defensible.

| # | Surface | Chapter | What it owns | Why it is load-bearing |
|---|---|---|---|---|
| 1 | Release-gate architecture | 02 | The gate policy — what evidence is required at each stage of a release, the thresholds, the rollback criteria, the surfaces in scope | Without it, every release argues its own gate. |
| 2 | Delegation contracts | 03 | The contracts with peer specialists that name what the eval program produces for them and what they produce for the eval program | Without them, peer teams route around eval or eval absorbs peer scope. |
| 3 | Build-vs-buy platform matrix | 04 | The decision on which eval platform components are bought vs. built, refreshed on a documented cadence | Without it, the platform is a rewrite every 18 months. |
| 4 | Incident-driven investment loop | 05 | The process that turns every P0 / P1 incident in scope into a regression fixture + alert + runbook diff, with a named owner and a deadline | Without it, the same regression recurs six months later and the team relearns it. |
| 5 | Product-side card slice | 06 | The composed model / system card slice — the eval evidence in a shape the governance peer can fold into the org-wide card and hand to internal review | Without it, the governance peer authors it from scratch and drift is inevitable. |
| 6 | Framework mapping | 07 | The mapping between the program's artefacts and the frameworks (NIST AI RMF Measure / Manage, EU AI Act GPAI provider obligations, ISO/IEC 42001 controls) the org is audited against | Without it, the audit walks in and finds no linkage between the eval evidence and the framework language. |

Two things the surfaces are **not**:

- They are not *another eval*. They are the shape by which every other module's eval work becomes a program.
- They are not *governance authorship*. Governance shape — the actual policy on high-risk use, the vendor agreement, the regulator interface — is owned by the peer track: `ai-evaluation-engineer` (Governance family, L25) and `ai-governance-analyst`. This module produces evidence in a shape the governance track can consume; it does not author governance.

### Vocabulary the module reuses

Six terms recur across the chapters. Use them consistently — the frameworks, the vendors, and the internal wikis each label them differently, and drift between labels is how a program's release-gate policy ends up meaning three different things depending on which team is reading.

- **App surface family.** A grouping of product surfaces sharing enough of an evaluation profile that one release-gate policy covers them — e.g., "customer-facing support agents," "internal knowledge-assistant surfaces," "developer copilot code surfaces." The unit at which chapter 02's gate architecture is defined. Not every surface is in every family; a surface belongs to at most one family.
- **Release gate.** A named check-point in a release process where a specific eval artefact must exist, be signed, and meet named thresholds before the release advances. Gates form a *set* per family (offline gate, canary gate, ramp gate, post-ship gate). Chapter 02 formalises the shape.
- **Delegation contract.** A written, versioned agreement between the eval program and a peer specialist that names what each side produces for the other, on what cadence, in what format, with what escalation shape. Chapter 03 walks the anatomy; the contracts in exercise 02 are the deliverable.
- **Program artefact.** An output the eval program owns that other tracks consume — the release-gate policy, the delegation contracts, the platform matrix, the incident investment ledger, the card slice, the framework mapping. Distinct from a *per-decision artefact* (a rubric run, a trade-off report) which the program *composes* from.
- **Investment ledger.** The running record of every regression fixture, alert, and runbook diff produced from an incident, with owner, incident reference, gate binding, and status. Chapter 05's central artefact.
- **Card slice.** The eval-program's contribution to the org-wide model / system card — one slice per app surface family — that the governance peer folds into the org-wide document. Chapter 06's central artefact. Not the full card; a slice.

### Where every prior module reads back

The program is not a new store hovering above the old ones. Every prior module's artefacts become the *evidence base* for one or more of the program surfaces.

| Prior module | Artefact | Where it appears in the program |
|---|---|---|
| mod-101 (product-shaped foundations) | Definition of what quality means per surface | The surface roster for chapter 02; the card slice's "what this surface does" section |
| mod-102 (trace instrumentation) | OTel-GenAI spans | The evidence base for every online gate; the delegation contract with `ai-infra-mlops` |
| mod-103 (trajectory & tool eval) | Trajectory scores | The gate at ramp for agentic surfaces; the card slice's tool-use section |
| mod-104 (LLM-as-judge) | Rubrics + calibration | The evidence base for offline gates; the delegation contract with `model-evaluation-engineer` on judge overlap |
| mod-105 (RAG eval) | Retrieval + faithfulness scores | The gate at offline for RAG surfaces; the card slice's retrieval section |
| mod-106 (CI gate) | Replay bundle + gate result | The offline gate mechanism chapter 02 references |
| mod-107 (online loop) | Online drift + rollback signals | The canary / ramp / post-ship gate mechanism |
| mod-108 (safety) | Findings + OWASP tags | The safety row of the gate; the delegation contract with `ai-risk-engineer` and `ai-infra-security` |
| mod-109 (human review) | Reviewer-labelled cases | The gate at offline; the delegation contract with `ai-risk-engineer` on adjudication |
| mod-110 (platform slice) | Store, schemas, lineage | The read layer for every gate; the build-vs-buy interface for chapter 04 |
| mod-111 (cost / latency / quality) | Multi-objective report | The evidence bundle every shipping decision the gate authorises publishes |

Nothing new is invented at program altitude. Every program artefact is a *composition* over prior module artefacts, plus the delegation contracts that keep the composition operating across teams.

### The three failure shapes this module prevents

Three modes a program-less eval team falls into. Each is prevented by a specific chapter.

**Failure 1 — the *ad-hoc gate* shape.** Every release argues its own bar. Six months in, the org cannot answer "what does 'ready to ship' mean for surfaces in the support family?" The eval team is invoked as a *consultation*, not as a gate. When a regression happens, the retrospective's `root cause` field says "we did not have a gate."

Prevented by chapter 02 — the release-gate policy is published, mapped to every surface in scope, and referenced by the CI and canary machinery.

**Failure 2 — the *sprawl* shape.** The eval team, being helpful, starts running red-team suites (the risk-engineer peer's remit), authoring judge model training data (the model-eval-eng peer's remit), and drafting policy language for the governance analyst. Six months in, the eval team's headcount is doing the peers' jobs, the peers do not exist because the eval team is doing them, and none of the peer roles get built out because nobody sees the gap.

Prevented by chapter 03 — the delegation contracts name what each peer produces, and the eval program refuses to permanently absorb peer scope (bootstrap is fine; permanent absorption is not).

**Failure 3 — the *platform churn* shape.** The eval platform is rewritten every 18 months because the last decision was made informally. Each rewrite loses history, invalidates old dashboards, and takes six months to rebuild parity. The eval team's real work — running gates, investing in incidents — is starved through every rewrite.

Prevented by chapter 04 — the build-vs-buy matrix is published, refreshed on a documented cadence, and defensible against evidence from the last cycle. Rewrites happen when the matrix says they should, not when a Slack thread wins.

Chapters 05, 06, and 07 layer on: the incident loop (so failure modes surface as reproducible regressions), the card slice (so the governance peer has a shape to consume), and the framework mapping (so external reviewers can trace the audit story end-to-end).

### The delegation triangle

The program does not stand alone. It sits at a three-way boundary between:

- **The AI Evaluation Engineer (L30, this module).** Owns the app-altitude eval program.
- **The Model-Development family peers.** `model-evaluation-engineer` (L30) owns model-altitude methodology (offline benchmark suites, judge-model training, dangerous-capability evals). The two roles co-own judge overlap and model-swap regression coverage.
- **The Governance family peers.** `ai-evaluation-engineer` (L25) owns release-assurance and the regulator interface; `ai-governance-analyst` owns the org-wide policy, third-party attestations, and enforcement escalation. The three roles co-own the card slice, the framework mapping, and the escalation shape when a shipped surface later trips a policy line.

The triangle recurs in every chapter. Chapter 02's release-gate architecture reads the model-altitude quality profile *from* the model-eval-eng peer and publishes the gate outcome *into* the governance peer's assurance evidence. Chapter 03 encodes the triangle as machine-readable delegation contracts. Chapter 06's card slice is authored by this module and consumed by the governance peer. Chapter 07's framework mapping is authored jointly with the governance peer.

An easy diagnostic for whether the triangle is intact: when a peer needs an artefact from you, can they name the file, the schema, and the SLA? If yes, the contract exists; if no, it needs to be authored.

### The program's rhythm

Program work happens on a longer cadence than app-altitude work. Practitioner-level eval work is *weekly* (a rubric run, a trade-off report, a gate outcome). Program work is *quarterly* (a policy refresh, a delegation contract review, a platform matrix revisit) with a *monthly* heartbeat (an investment ledger review, a card slice diff, a framework-mapping check).

Concretely, the rhythm the module builds:

- **Weekly.** Practitioner eval work (owned by peers and embedded eval engineers). Program-owner attends the on-call handoff to hear what shipped and what did not, and the incident review to enqueue investment items.
- **Monthly.** Investment ledger review (are the fixtures / alerts / runbook diffs from last month's incidents complete or on track?). Card slice diff review with the governance peer (what changed on the surfaces the program owns?). Framework-mapping check (any framework updates that require re-linking?).
- **Quarterly.** Release-gate policy refresh (are the thresholds still where they should be?). Delegation contract review with every peer (are the contracts working; where is friction?). Platform matrix status check (is any component's build-vs-buy call still valid?).
- **Annually.** Platform matrix full revisit (the build-vs-buy decision, from scratch, against last year's evidence). Program-scope review (any new app surface families? any families exiting scope? any peer roles that were bootstrapped this year that should now be spun out?).

The rhythm is what makes the program *ownable* rather than a one-shot document. Chapter 02 and chapter 04 pin the cadences; chapter 05 anchors the monthly investment review.

### What the program is *not*

Two things worth naming explicitly to keep scope crisp.

- **The program is not the governance function.** The governance function (peer track: `ai-evaluation-engineer` L25, `ai-governance-analyst`) owns policy authorship, third-party attestations, regulator interface, and the org-wide model/system card. This module produces the *inputs* the governance function consumes — release-gate outcomes, incident investment records, per-surface card slices, framework mappings — in shapes the governance function can consume without rewrites. Chapter 06 and chapter 07 formalise the hand-off.
- **The program is not headcount management for eval.** Deciding how many eval engineers to hire, how they sit in the org (embedded vs. central), and how to career-ladder them is a management concern this module does not walk. It informs those decisions (the release-gate architecture implies staffing shape; the delegation contracts imply peer boundaries) but does not author them. Manager-track guidance sits outside this curriculum.

### The end-of-module posture

By the end of this module you own the AI evaluation program. Concretely, you can:

- Show a **release-gate policy** for at least one app surface family — the surfaces in scope, the gates and thresholds per stage, the rollback criteria, the CI / canary / ramp machinery bindings, and the policy signer.
- Show a **delegation contract set** with every peer specialist the program interfaces with — machine-readable, versioned, referenced from the ticket templates used to escalate.
- Show a **build-vs-buy platform matrix** — every component of the eval platform on the row, with the current decision (buy / build / hybrid), the vendor short-list, the residency / extensibility / vendor-lock trade-offs, the cost line, and the next refresh date.
- Show an **investment ledger** — every P0 / P1 in scope from the last quarter, mapped to a regression fixture, an alert, and a runbook diff, with owner and status.
- Show a **product-side card slice** for at least one app surface family — signed by product and eng, folded into the org-wide model/system card by the governance peer with zero rewrites.
- Show a **framework mapping** — every program artefact tagged with the NIST AI RMF, EU AI Act, and ISO/IEC 42001 controls it evidences, in a shape the audit walk-through can consume in a single sitting.
- Escalate cleanly to the peer tracks — the model-eval-eng peer for capability-profile refreshes and judge-methodology issues; the risk-eng peer for harm-model and red-team-data refreshes; the governance peers for policy or regulator questions; the infra-security peer for judge supply-chain or adversarial-eval questions; the mlops / ml-platform peers for CI or platform integration issues.

The chapters build these one at a time. Chapter 02 opens the release-gate architecture — the surface every subsequent chapter's evidence eventually feeds through.

## Summary

- The module builds the **program-owner posture** the previous eleven modules feed into — the release gates, the delegation contracts, the platform decisions, the incident investment loop, the product-side card slice, and the framework mapping.
- The **program altitude** sits above the app altitude — it reads down from the mod-101 – mod-111 artefacts and publishes up into the org's release, governance, and regulator-facing surfaces. It does not re-produce the app-altitude artefacts; it *composes* them.
- **Six load-bearing surfaces**: release-gate architecture (ch. 02), delegation contracts (ch. 03), build-vs-buy platform matrix (ch. 04), incident-driven investment loop (ch. 05), product-side card slice (ch. 06), framework mapping (ch. 07).
- **Six vocabulary terms**: **app surface family**, **release gate**, **delegation contract**, **program artefact**, **investment ledger**, **card slice**.
- **Three failure shapes** the module prevents: the *ad-hoc gate* shape (every release argues its own bar), the *sprawl* shape (eval absorbs peer scope), the *platform churn* shape (rewrite every 18 months).
- The **delegation triangle** — this module + `model-evaluation-engineer` (L30) + the Governance family peers (`ai-evaluation-engineer` L25, `ai-governance-analyst`) — recurs in every chapter and is machine-readable in chapter 03.
- The **program's rhythm** is weekly at the practitioner level, monthly for the investment ledger and card slice diff, quarterly for the release-gate policy and delegation contract review, annually for the platform matrix full revisit.
- The program is **not the governance function** (owned by the peer track) and **not headcount management** (owned by the manager track).

Chapter 02 opens the release-gate architecture — the policy every subsequent chapter's artefacts eventually route through.
