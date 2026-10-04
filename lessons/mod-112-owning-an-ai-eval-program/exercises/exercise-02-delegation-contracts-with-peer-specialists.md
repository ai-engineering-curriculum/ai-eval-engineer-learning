# exercise-02: Delegation Contracts With Peer Specialists

**Estimated effort:** 2 hours

## Objective

Author the **six delegation contracts** the chapter 03 anatomy describes — one per peer specialist the eval program interfaces with — and bind each contract's artefact list to a downstream chapter so every artefact has a named producer and a named consumer. The contracts turn ad-hoc "ask the peer when you need it" relationships into reviewable, versioned agreements, with a quarterly refresh cadence and an escalation shape when either side misses.

By the end of the exercise the eval program's peer relationships are a `programs/delegation/` tree: six YAML contracts (model-eval-eng, risk-eng, ai-evaluation-engineer-governance, governance-analyst, infra-security, mlops / ml-platform), one ticket template per interface channel, a joint-review calendar entry per contract, and a bootstrap-state audit that honestly documents which peer functions are fully in place vs. being stood up. The sprawl failure shape from chapter 01 — eval absorbing peer scope — becomes observable at the quarterly review rather than silently structural.

## Prerequisites

- Chapters 01 and 03 of this module.
- Exercise 01 — the release-gate architecture is the primary consumer of several peer artefacts; the contracts bind to it.
- Access to each peer counterpart (even if the "peer" today is one engineer wearing multiple hats). The contract names the peer owner; signing the contract requires a conversation with that person.
- A code-hosting repo for the eval program; the `programs/delegation/` tree lands there alongside `programs/gates/` from exercise 01.
- A shared ticket tracker your org uses (Jira, Linear, GitHub Projects, Shortcut) with templating support — the escalation and request-shape ticket templates live there.
- A shared calendar the quarterly reviews land on. If the peer is in a different org (e.g., the ML platform team is a different reporting line), the contract covers who schedules.
- Python 3.11+; `pydantic` or `jsonschema` for the contract validator; `pyyaml` for the YAML load / dump.

## Set-up

1. Create the delegation directory:

   ```
   programs/delegation/
   ├── contracts/
   │   ├── model-evaluation-engineer.yaml
   │   ├── ai-risk-engineer.yaml
   │   ├── ai-evaluation-engineer-governance.yaml      # the L25 release-assurance peer
   │   ├── ai-governance-analyst.yaml
   │   ├── ai-infra-security.yaml
   │   └── ai-infra-mlops-ml-platform.yaml             # or split if your org has two roles
   ├── templates/
   │   ├── escalation-to-model-eval-eng.md
   │   ├── escalation-to-ai-risk-engineer.md
   │   ├── escalation-to-governance.md
   │   ├── escalation-to-ai-infra-security.md
   │   ├── escalation-to-mlops-ml-platform.md
   │   └── joint-review-agenda.md
   ├── bootstrap/
   │   ├── bootstrap-audit.md                           # honest write-up of which peer functions exist
   │   └── handoff-plans/
   │       └── <peer>-handoff.md                        # one per absorbed-today-handoff-later artefact
   ├── schema/
   │   └── contract.schema.json
   ├── validate/
   │   ├── validate_contract.py                         # schema + semantic checks
   │   ├── validate_consumers.py                        # every artefact has a named consumer
   │   └── validate_bootstrap_handoff.py                # every absorbed artefact has a hand-off date
   ├── history/
   │   ├── 2026-Q4-initial.md
   │   └── reviews/
   │       └── .gitkeep
   └── README.md
   ```

2. Author `schema/contract.schema.json` to enforce the chapter 03 eight-section anatomy. Required blocks:
   - `peer` (string, from the known-peer enum)
   - `version` (string, e.g. `2026-Q4`)
   - `signed` (list of objects with `role`, `name`, `date`)
   - `review_cadence`, `next_review`
   - `scope.covers` (non-empty list)
   - `scope.does_not_cover` (non-empty list; the exclusions matter)
   - `what_the_peer_produces` (list of artefact objects: `artefact`, `shape`, `cadence`, `lands_in`, `consumed_by` with ≥ 1 downstream chapter / exercise reference)
   - `what_the_eval_program_produces` (symmetric list; every contract has this side populated)
   - `interface_channels` (list of channels: `kind`, `template`, `tracker`, SLAs)
   - `escalation.eval_program_owes_peer_late` and `escalation.peer_owes_eval_program_late` (triggers + actions)
   - `bootstrap_state.peer_exists`, `peer_bootstrapping`, `eval_program_bootstrapping_peer`, `bootstrap_plan` (either `n/a` or a readable path)

3. Fill each contract. Below is the shape for `contracts/model-evaluation-engineer.yaml` with placeholders — adapt to your org's reality for each of the six:

   ```yaml
   peer: model-evaluation-engineer
   peer_family: Model-Development
   peer_level: 30
   version: 2026-Q4
   signed:
     - role: eval_program_owner
       name: <you>
       date: 2026-10-XX
     - role: peer_owner
       name: <peer counterpart or placeholder with bootstrap note>
       date: 2026-10-XX
   review_cadence: quarterly
   next_review: 2027-01-XX

   scope:
     covers:
       - Model-altitude capability profile requests scoped to shipping decisions on {families_in_scope}
       - Judge-overlap diagnostics on rubrics used by {families_in_scope}
       - Model-swap regression coverage planning for {families_in_scope}
     does_not_cover:
       - Peer's dangerous-capability eval program (owned end-to-end by peer)
       - App-altitude cohort authoring (owned end-to-end by eval program)
       - Vendor procurement decisions (owned by peer with input from eval program)

   what_the_peer_produces:
     - artefact: capability_profile_snapshot
       shape: peer's standard schema; JSON per model_snapshot
       cadence: monthly refresh; on-demand within 5 business days for a shipping decision
       lands_in: mod-110 store, key `capability_profiles.model_snapshot`
       consumed_by:
         - chapter 02 offline gate for model_snapshot and model_family change classes
         - chapter 06 card slice § 3 (model capability profile)
     - artefact: judge_overlap_diagnostic
       shape: paired-comparison rubric outputs on peer's held-out set
       cadence: quarterly; on-demand for a rubric methodology question
       lands_in: mod-110 store, key `judge_diagnostics.rubric_id`
       consumed_by:
         - mod-104 chapter 05 judge calibration
         - chapter 06 card slice § 3 (judge methodology)
     - artefact: model_swap_regression_coverage_plan
       shape: Markdown doc per family transition
       cadence: on-demand per transition
       lands_in: `programs/gates/<family>/history/model-swap-<id>.md`
       consumed_by:
         - chapter 02 release-gate policy per surface

   what_the_eval_program_produces:
     - artefact: cohort_regression_report
       shape: mod-111 ch. 04 delta report on peer's benchmark cohorts
       cadence: on-demand when app-altitude finds a regression attributable to model altitude
       lands_in: peer's tracker; referenced from mod-110 `escalations` table
     - artefact: judge_overlap_result
       shape: eval program's paired-comparison result against peer's diagnostic
       cadence: on receipt of peer's diagnostic (within 10 business days)
       lands_in: mod-110 store; peer's tracker
     - artefact: production_usage_summary
       shape: monthly summary of model snapshots in production per surface
       cadence: monthly
       lands_in: peer's tracker, read-only shared folder

   interface_channels:
     - kind: escalation_ticket
       template: templates/escalation-to-model-eval-eng.md
       tracker: <jira project | linear team | GH project>
       sla_ack: 2 business days
       sla_scoping: 5 business days
     - kind: joint_review
       cadence: monthly
       duration: 30 min
       agenda: templates/joint-review-agenda.md

   escalation:
     eval_program_owes_peer_late:
       trigger: any owed artefact > SLA by > 3 business days
       action: eval program owner acknowledges within 1 bd; add to next quarterly review
     peer_owes_eval_program_late:
       trigger: any owed artefact > SLA by > 3 business days
       action: escalate to peer owner and eval program owner's manager; add to next quarterly review; if 2 consecutive quarters miss, escalate to cross-team engineering leadership

   bootstrap_state:
     peer_exists: yes
     peer_bootstrapping: no
     eval_program_bootstrapping_peer: no
     bootstrap_plan: n/a
   ```

4. Author one ticket template per interface channel in `templates/`. The template includes the required fields the receiving side needs on day one: the shipping decision context, the lineage keys, the surface, the deadline, the acceptance shape, the contact. The templates reduce the "what did you want again?" ping-pong.

5. Author `bootstrap/bootstrap-audit.md` — a brutally honest page per peer answering: Does the peer function exist in your org? Who is the counterpart (if any)? Which artefacts in the contract is the eval program temporarily producing because the peer is not in place? What is the hand-off plan and timeline?

   For every absorbed artefact, author `bootstrap/handoff-plans/<peer>-handoff.md` with the acceptance criteria the peer owner will meet to take the artefact over, the target hand-off date, and the escalation path if the date slips.

6. Author `templates/joint-review-agenda.md` as the standing agenda per contract review:
   - Artefact production + consumption since last review (per-side)
   - SLA breaches (if any)
   - Interface-channel friction (templates fit for purpose?)
   - Escalation events (triggered? resolved?)
   - Bootstrap-plan progress (hand-off dates on track?)
   - Contract drift (any new work either side is doing not in the contract?)
   - Next-review date + action items

## Requirements

Produce a PR against your eval-program repo that adds:

1. **Six signed contracts in `contracts/`** — one per peer, each conforming to the schema, each with the eight sections filled; symmetric production (never one-way); scope exclusions explicit; consumers of every peer artefact referenced by chapter number and (where applicable) by exercise.
2. **`schema/contract.schema.json`** — the JSON schema enforcing the eight-section anatomy; validation errors are human-readable and name the missing field.
3. **`validate/validate_contract.py`** — schema + semantic validation:
   - Both `scope.covers` and `scope.does_not_cover` are non-empty.
   - Every artefact in `what_the_peer_produces` has at least one `consumed_by` entry (consumer-less artefacts are a chapter 03 anti-pattern).
   - Every artefact in `what_the_eval_program_produces` has at least one `lands_in` entry.
   - `bootstrap_state` flags are internally consistent — if any `*_bootstrapping` flag is `yes`, `bootstrap_plan` points to a real file.
   - `next_review` is in the future.
4. **`validate/validate_consumers.py`** — cross-contract check:
   - Every artefact a chapter is documented to consume (per the chapter cross-references) is produced by at least one peer contract or by the eval program's chapter 02 / 05 / 06 artefacts.
   - Every artefact produced by the eval program for a peer appears as a consumer in that peer's contract.
5. **`validate/validate_bootstrap_handoff.py`** — every absorbed artefact has a `bootstrap/handoff-plans/<peer>-handoff.md` file with a hand-off date; absent files or past-date hand-offs without renewal fail the check.
6. **Ticket templates in `templates/`** — one per peer per interface channel. The template includes a `front-matter` block the escalation ticket system can parse (surface, change class, deadline, acceptance shape) and a free-form body for the request specifics.
7. **`bootstrap/bootstrap-audit.md`** — the per-peer honest audit; names the counterpart (or lack thereof); names the absorbed artefacts; names the hand-off dates; references the hand-off plans.
8. **Hand-off plan docs** — one per absorbed artefact; acceptance criteria the peer will meet to take over; owner; target date; escalation path on slip.
9. **CI integration** — the three validators run on every PR touching `programs/delegation/`. A removed consumer, a missing bootstrap plan, or an in-the-past `next_review` fails the CI.
10. **`history/2026-Q4-initial.md`** — the authoring note: which peer functions are in place today, which are being stood up, which are entirely absent and absorbed by the eval program, and the quarterly-review calendar entries per peer.
11. **`README.md`** — the directory's self-standing overview: the six peers, the current state (`in_place` / `bootstrap` / `absorbed`), the next-review schedule, the contact matrix.
12. **Calendar entries** — one quarterly-review entry per peer on the shared calendar, with the eval program owner as owner and the peer counterpart (or placeholder) invited. The entry links to the contract and the standing agenda.

## Starter guidance

- **Author all six, even the ones your org does not have yet.** The chapter 03 point about bootstrap-state is that the honest acknowledgement is what makes the sprawl visible. A peer function that does not exist is still a contract — the contract names what the eval program is temporarily doing and the plan to spin out. Skipping the contract because "we don't have a risk-eng peer yet" is how sprawl becomes structural.
- **The `does_not_cover` list is as important as `covers`.** Most drift happens in the "well, this feels related to your work" grey zone. Spend real time on exclusions. If you cannot name five exclusions per peer, you have not thought about the boundary.
- **Symmetric contracts prevent extractive relationships.** Every peer contract has `what_the_eval_program_produces` populated with real artefacts the peer actually uses. A one-way contract (peer gives, eval takes) silently erodes because the peer under-invests. Chapter 03 is explicit on this.
- **Consumers by chapter reference, not by prose.** `consumed_by: chapter 02 offline gate for model_snapshot change classes` is enforceable; `consumed_by: the eval program's work` is not. The `validate_consumers.py` validator crosschecks against chapter references so a dangling consumer is caught.
- **Bootstrap is time-boxed; absorption is not.** The chapter 03 bootstrap-state clause is honest only if every absorbed artefact has a hand-off date and the date is reviewed each quarter. A hand-off date slipped twice without new evidence for the timeline is a signal to escalate to engineering leadership — not to extend the date silently.
- **The L25 release-assurance peer and the governance analyst are two contracts.** They share a lot of shape; author them as two files, not one. The split lets each refresh independently as the roles clarify; collapsing them hides the ownership boundary between release-assurance and policy / regulator interface.
- **The MLOps and ML-platform peers are often one contract.** Many orgs have one person covering both roles; one contract is fine. If your org has two distinct peers, split the file — the contract shape is the same.
- **The templates are what make the escalation fast.** A peer with ten eval-program tickets filed through a half-formed template spends their day asking for context. A peer with ten tickets filed through a well-templated form answers them. The templates matter more than they look.
- **The quarterly review is the mechanism.** The contract document is static; the review is where the contract stays alive. Put the recurring calendar entries in place now, not later. If the first review is three months out and nobody has it on their calendar, the chance of it happening drops to zero.
- **Private-sector reality check**: in a young org, several peers will be `peer_exists: no` and the eval program will be absorbing. The honest audit is the first step; the hand-off plans are the second; the hiring conversation with engineering leadership (triggered by the audit) is the third.

## Acceptance criteria

You are done when:

- Six contracts exist in `contracts/`, each validates against the schema, each has the eight sections populated (header, scope, what the peer produces, what the eval program produces, interface channels, escalation, bootstrap state, review cadence).
- Every peer-produced artefact has at least one `consumed_by` entry referencing a specific chapter / exercise; `validate_consumers.py` passes without warnings about dangling consumers.
- Every artefact the eval program is absorbing today has a hand-off plan in `bootstrap/handoff-plans/`; `validate_bootstrap_handoff.py` passes.
- Six ticket templates exist in `templates/` (one per peer) plus the joint-review agenda; the template front-matter is parseable.
- `bootstrap/bootstrap-audit.md` honestly names each peer's state (`in_place`, `bootstrap`, `absorbed`) with a one-paragraph justification per peer.
- CI runs the three validators on every PR touching `programs/delegation/`; a deliberate break (remove a consumer; delete a hand-off plan; set `next_review` to the past) fails the CI.
- Quarterly-review calendar entries exist on the shared calendar for all six peers, with the eval program owner as owner and the first review within the next 90 days.
- The `README.md` reads as a self-standing overview — a peer counterpart opening the file for the first time finds the contract they signed, the next review date, and the contact for the eval-program side.
- `history/2026-Q4-initial.md` names which contracts are green (`peer_exists: yes`, symmetric production flowing), which are yellow (bootstrap in progress), and which are red (peer absent, absorption underway with a credible hand-off date) — the colour-coding makes the sprawl state visible at a glance.

## Stretch goals

- **Contract-diff review.** On each quarterly refresh, generate a diff of the contract versus the previous version with annotated commentary per section. The diff is what the joint review walks through.
- **Artefact flow visualiser.** Parse all six contracts and emit a Graphviz / Mermaid diagram of the artefact flow — which peer produces which artefact for which chapter / exercise, which eval-program artefact lands with which peer. The diagram is a one-page view of the delegation triangle.
- **SLA breach dashboard.** Instrument the escalation-ticket templates with timestamps; compute mean / p95 SLA ack and scoping times per peer per quarter; feed the dashboard into the quarterly review.
- **Bootstrap-plan burn-down.** For each absorbed artefact, track the acceptance criteria completion over time; the burn-down chart is what the escalation to engineering leadership references.
- **Contract-renewal automation.** When `next_review` is 30 days out, the system auto-opens a ticket per peer with the joint-review-agenda template pre-filled. Reduces the ceremony friction.
- **Cross-track contract linking.** Link each contract to the peer's track's `README.md` (e.g., `model-evaluation-engineer-learning`, `ai-risk-engineer-learning`). The link makes the peer's level and remit discoverable to a new reader; useful when the counterpart changes.
- **Audit-ready contract bundle.** For each contract, generate a per-quarter audit bundle: the contract, the quarter's escalation events, the quarter's artefact production summary, the signer attestations. The bundle is what a governance peer hands to a third-party auditor.

## What this exercise does *not* cover

You are not authoring the release-gate architecture (exercise 01), the build-vs-buy matrix (exercise 03), the incident investment ledger (exercise 04), or the card slice generator (exercise 05). You are shipping the *six delegation contracts* — the shape that keeps peer specialists working with the eval program on a reviewable cadence and stops the chapter 01 sprawl failure before it becomes structural.
