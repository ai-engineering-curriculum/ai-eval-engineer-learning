# exercise-01: Release Gate Architecture For One App Family

**Estimated effort:** 3 hours

## Objective

Author and publish a **release-gate architecture** for one app surface family under your scope — the chapter 02 artefact every subsequent exercise's work composes over. You will commit a signed, versioned policy document that names the surfaces in the family, the four-gate topology (offline, canary, ramp, post-ship), the gate map per surface × change class, every threshold with its provenance, and a paired rollback criterion per threshold. The policy ships with release runbooks, a waiver / emergency-path mechanism, and a quarterly-refresh calendar entry.

By the end of the exercise the family's release process is the policy — the CI machinery enforces the offline gate from the file, the canary / ramp machinery reads the gate map, and the post-ship rollback triggers fire from `rollback_criteria.yaml`. "What does ready-to-ship mean for this family?" has a single documented answer, and any release that tries to bypass the gates fails loudly rather than quietly.

## Prerequisites

- Chapters 01 and 02 of this module.
- One **app surface family** you can own the policy for — at least one in-production surface, preferably two to five surfaces that share an evaluation profile (e.g., `customer_facing_support_agents`, `internal_knowledge_assistant`, `developer_copilot_code`). The family does not need to be the whole org, but it needs real surfaces with real traffic.
- Mod-106 (CI-gated eval) at least at the replay-bundle stage — your offline gate reads the mod-106 artefact.
- Mod-107 (online eval and regression) at least at the online-loop metric stage — your canary / ramp / post-ship gates read mod-107 metrics. If mod-107 is still being stood up, the gate shape is the same but the enforcement is stub-logged instead of flag-flipped (noted in the policy's `bootstrap_state` section).
- Mod-108 (safety and guardrails) findings for every surface in the family — the offline gate's safety evidence row cites these.
- Mod-111 (cost / latency / quality trade-off) report shape — the offline gate references the pre-registered rule and the mod-107 rollback trigger.
- Access to the family's existing release process (even if ad-hoc) and the on-call rotation for each surface — the runbook names escalation contacts.
- A code-hosting repo for the eval program (`eval-program/` or similar) with CI access, so the policy document and the `programs/gates/<family>/` tree land somewhere the CI can read.
- Python 3.11+; a YAML / schema library (`pydantic` or `jsonschema`) for the policy validator; `markdown-it` or similar for the runbook renderer if you want HTML output.

## Set-up

1. Create the family's program directory:

   ```
   programs/gates/<family>/
   ├── policy.yaml                       # the gate policy — surfaces, gate map, thresholds
   ├── rollback_criteria.yaml            # rollback triggers + runbooks (referenced by policy.yaml)
   ├── change_classes.md                 # what each change class means for this family
   ├── evidence_shape.md                 # which artefact each gate reads; where it lives in mod-110
   ├── runbooks/
   │   ├── <surface>-release.md          # one per surface
   │   ├── <surface>-<rollback>-rollback.md
   │   └── waiver-process.md
   ├── history/
   │   ├── 2026-Q4-initial.md            # this exercise's authoring note
   │   └── waivers/
   │       └── .gitkeep
   ├── schema/
   │   ├── policy.schema.json            # JSON schema for policy.yaml
   │   └── rollback.schema.json          # JSON schema for rollback_criteria.yaml
   ├── validate/
   │   ├── validate_policy.py            # schema + semantic validation
   │   ├── validate_signatures.py        # signer presence + role coverage check
   │   └── validate_runbook_links.py     # every rollback trigger points to a readable runbook
   ├── simulate/
   │   ├── fake_release.py               # dry-run a candidate release through the gate map
   │   └── fixtures/
   │       ├── passing_release.yaml
   │       └── failing_release.yaml
   ├── bind/
   │   ├── ci_binding.md                 # how the mod-106 CI reads policy.yaml
   │   ├── canary_binding.md             # how the mod-107 canary machinery reads the gate map
   │   └── rollback_binding.md           # how mod-107 alerts reference rollback_criteria.yaml
   └── README.md                         # signers, cadence, escalation contacts, refresh date
   ```

2. Fill `policy.yaml` with the full family shape. The example skeleton (adapt the surfaces, change classes, and thresholds to your family):

   ```yaml
   family: customer_facing_support_agents
   version: 2026-Q4
   signed:
     - role: eval_program_owner
       name: <you>
       date: 2026-10-XX
     - role: product_owner
       name: <product owner>
       date: 2026-10-XX
     - role: engineering_lead
       name: <eng lead>
       date: 2026-10-XX
     - role: safety_peer                 # ai-risk-engineer or governance peer per org shape
       name: <peer>
       date: 2026-10-XX
     - role: governance_peer             # required if any surface is in regulated scope
       name: <governance peer>
       date: 2026-10-XX
   review_cadence: quarterly
   next_review: 2027-01-XX
   escalation:
     program_owner: <contact>
     on_call_rotation: <pagerduty schedule or similar>

   surfaces:
     - name: support_bot
       owner: <product + eng pair>
       traffic_class: high                # low | medium | high — informs ramp shape
       regulated: no
       change_classes:
         prompt_minor:
           offline: required
           canary: { duration: 4h,  share: 5 }
           ramp:   [ 25, 50, 100 ]
           ramp_dwell: 24h
           post_ship: required
         prompt_major:
           offline: required
           canary: { duration: 8h,  share: 5 }
           ramp:   [ 10, 25, 50, 100 ]
           ramp_dwell: 48h
           post_ship: required
         model_snapshot:
           offline: required
           canary: { duration: 24h, share: 5 }
           ramp:   [ 5, 25, 50, 100 ]
           ramp_dwell: 72h
           post_ship: required
         model_family:
           offline: required
           canary: { duration: 48h, share: 5 }
           ramp:   [ 5, 10, 25, 50, 100 ]
           ramp_dwell: 5d
           post_ship: required
         retriever_index:
           offline: required
           canary: { duration: 24h, share: 5 }
           ramp:   [ 25, 50, 100 ]
           ramp_dwell: 48h
           post_ship: required
         tool_added:
           offline: required
           canary: { duration: 24h, share: 5 }
           ramp:   [ 10, 25, 50, 100 ]
           ramp_dwell: 48h
           post_ship: required
         guardrail_change:
           offline: required
           canary: { duration: 12h, share: 5 }
           ramp:   [ 25, 50, 100 ]
           ramp_dwell: 24h
           post_ship: required
     - name: internal_support_lookups
       owner: <product + eng pair>
       traffic_class: low
       regulated: no
       change_classes:
         prompt_minor:
           offline: required
           canary: skip
           ramp:   [ 100 ]
           post_ship: required
         # ... other change classes abbreviated similarly

   gates:
     offline:
       enforcement: hard_block            # merge is blocked
       evidence:
         - mod_106_replay_bundle
         - mod_108_safety_report
         - mod_111_tradeoff_report
       thresholds:
         - metric: faithfulness.mean
           cohort: regulated
           bound: ">= 4.1"
           provenance:
             kind: product_committed
             source: PR-Support-2026-Q2.md
             owner: product_owner
         - metric: cohort_preservation.worst_delta
           cohort: all
           bound: ">= -0.8"
           provenance:
             kind: incumbent_relative
             source: mod-111 ch. 03 cohort-preservation contract
         - metric: owasp_llm_top10.regression_count
           cohort: all
           bound: "== 0"
           provenance:
             kind: safety_mandated
             source: ai-risk-engineer harm model 2026-Q3
             owner: ai_risk_engineer
       signers: [eng_lead, eval_program_owner]
     canary:
       enforcement: soft_block_auto_rollback
       evidence:
         - mod_107_canary_window
         - mod_108_in_prod_findings
       thresholds:
         # ... as above, scoped to canary window
       escalation_borderline: on_call + eval_program_owner
     ramp:
       enforcement: soft_block_hold
       evidence:
         - mod_107_ramp_step_window
         - mod_111_cost_latency_delta
       thresholds:
         # ...
     post_ship:
       enforcement: rolling_rollback_trigger
       evidence:
         - mod_107_online_loop
         - monthly_card_slice_diff
       thresholds_source: rollback_criteria.yaml

   waiver:
     signers_required: [eval_program_owner, product_owner, engineering_lead]
     expiry_max_days: 30
     follow_up: investment_ledger_entry
     log_path: history/waivers/
   emergency_path:
     name: emergency_release
     gates_required: [offline]            # canary / ramp skipped
     post_ship_window: 24h                # tightened monitoring
     count_cap_per_quarter: 2             # above this, refresh triggered
   ```

3. Fill `rollback_criteria.yaml` with one entry per paired threshold (see chapter 02's shape). Each entry names the mod-107 metric, cohort, window, threshold expression (expressed as a delta against baseline), action (`flag_flip`, `router_tier_restrict`, `ramp_down`), the flag or router policy, grace period, runbook path, and escalation contacts. Safety rollbacks have `grace_period: none`; quality rollbacks typically have a `grace_period: 24h_since_previous_rollback`.

4. Author `change_classes.md` with a one-paragraph description per change class — what counts as `prompt_minor` vs `prompt_major` for this family, what counts as `model_snapshot` vs `model_family`, what a `retriever_index` change includes, etc. The release machinery reads this doc; ambiguity here is how a release ends up in the wrong gate shape.

5. Author `evidence_shape.md` — for each gate, which artefact is read, where it lives in mod-110 (the table + the key shape), who authors it, and the staleness tolerance. This is the "where does the evidence come from" reference the on-call reads during an incident.

6. Author one release runbook per surface in `runbooks/<surface>-release.md` and one rollback runbook per rollback trigger in `runbooks/<surface>-<rollback>-rollback.md`. The release runbook covers pre-release / offline / canary / ramp / post-ship steps per chapter 02; the rollback runbook covers the manual-fire path, escalation contacts, incident-declaration criteria, and investment-ledger entry (chapter 05).

7. Author `runbooks/waiver-process.md` describing how a waiver is filed, who signs, what the expiry rules are, and what follow-up the waiver commits to.

## Requirements

Produce a PR against your eval-program repo that adds:

1. **`policy.yaml`** — the signed policy with the full surface roster, gate map per surface × change class, four-gate topology with evidence and thresholds (each threshold carrying its provenance), waiver and emergency-path shape, signers for every required role, and a `next_review` date.
2. **`rollback_criteria.yaml`** — every post-ship rollback trigger, each with mod-107 metric, cohort scope, window, threshold expression, action, flag / policy reference, grace period, runbook path, and escalation contacts. Every rollback trigger named in `policy.yaml`'s post-ship gate section must have a matching entry.
3. **`change_classes.md`** — the written definition of every change class the gate map differentiates on; concrete examples per class so an engineer labelling a PR can self-serve.
4. **`evidence_shape.md`** — the per-gate evidence reference; mod-110 table + key shape per artefact; staleness tolerance; authoring owner.
5. **`runbooks/*`** — one release runbook per surface; one rollback runbook per rollback trigger; the waiver-process runbook. Every runbook names escalation contacts, the manual-command paths, and the investment-ledger-entry step for incident follow-up.
6. **`schema/policy.schema.json`** and **`schema/rollback.schema.json`** — JSON schemas for the two YAML files. The schemas enforce required fields, enum values (`traffic_class`, `enforcement`), threshold-provenance presence, and signer-role coverage.
7. **`validate/validate_policy.py`** — schema validation + semantic checks:
   - Every surface has a defined change-class map.
   - Every threshold has a `provenance` block with a non-empty `kind` and `source`.
   - Every regulated surface has a `governance_peer` signer.
   - `next_review` is in the future (not in the past).
   - Every change class named in a surface's map is defined in `change_classes.md` (grep check).
8. **`validate/validate_signatures.py`** — checks that every required role has a signer (eval_program_owner, product_owner, engineering_lead, safety_peer, and governance_peer if any surface is `regulated: yes`). Dates present on every signer. Signer date not earlier than the previous version's date.
9. **`validate/validate_runbook_links.py`** — every rollback-criteria entry's `runbook` path exists and is readable; every release-runbook referenced from `policy.yaml` exists.
10. **`simulate/fake_release.py`** — dry-runs a candidate release through the gate map. Input: a release manifest naming the surface, the change class, and the evidence bundle paths. Output: per-gate pass / fail / borderline verdict with the thresholds walked and the provenance cited; the ramp plan the machinery would execute; the rollback triggers that would arm post-ship. Two fixtures (`passing_release.yaml`, `failing_release.yaml`) exercise both outcomes.
11. **`bind/*`** — the three binding docs that describe how the policy connects to real machinery. If the machinery is already in place, the binding is live (named commits / configs that read the policy). If the machinery is still being stood up (bootstrap state), the binding doc describes the stub shape and names the hand-off date.
12. **`history/2026-Q4-initial.md`** — the authoring note. Explain the family scope, the surfaces chosen, the threshold-provenance sourcing, any bootstrap-state deviations from chapter 02, and the next-refresh calendar entry.
13. **`README.md`** — one-page family overview: scope, surfaces, signers, cadence, escalation contacts, and pointers to every artefact in the directory. The file an on-call engineer opens first.
14. **CI integration** — a CI job (`.github/workflows/`, `.gitlab-ci.yml`, or equivalent) that runs the three validators on every PR touching `programs/gates/<family>/`. A failing validator blocks the PR.
15. **A demonstration dry-run** — run `simulate/fake_release.py` against both fixtures; capture the output in `simulate/output/`; link the output from the authoring note. The demonstration shows a `prompt_major` passing release (gates clean, ramp plan generated, rollback triggers armed) and a `model_family` failing release (one threshold fails with a provenance citation, release blocked, rollback plan would not fire because the release never advanced).

## Starter guidance

- **Pick one family. Resist the urge to cover all of them.** The chapter's warning about "one org-wide policy" is the single biggest trap. A policy that tries to cover support agents + internal knowledge assistants + developer copilots at once becomes an unreadable matrix; a family-scoped policy is defensible in one sitting. Author one family now; stand up others later.
- **Threshold provenance is the exercise's central discipline.** The temptation is to type `>= 4.1` and move on. Resist it. Every threshold cites a source — a product doc (`product_committed`), a mod-111 cohort-preservation contract (`incumbent_relative`), the risk-eng peer's harm model (`safety_mandated`), or the governance peer's regulator-derived requirements (`regulator_derived`). A validator failure on missing provenance is a feature, not a bug.
- **Change classes are the gate map's spine.** Spend real time on `change_classes.md`. If `prompt_major` is "a bigger prompt change than `prompt_minor`," the gate map is unenforceable. Author concrete examples per class — "adds a new tool to the toolset" is `tool_added`; "swaps `claude-sonnet-4-6` for `claude-opus-4-7`" is `model_snapshot`; "swaps Anthropic for OpenAI" is `model_family`.
- **The policy is document + code, not a wiki page.** The release machinery must read `policy.yaml`; the on-call must read the Markdown runbooks. A policy that lives only in a wiki is the chapter 02 anti-pattern "gate-as-consultation." The CI binding doc is where you prove the enforcement is real.
- **Signers are roles, not people.** `eval_program_owner`, `product_owner`, `engineering_lead` — the role is what matters. People move; roles persist. The validator checks the role, not the name.
- **Rollback triggers without runbooks are a chapter 02 anti-pattern.** Every entry in `rollback_criteria.yaml` must point to a readable runbook; the `validate_runbook_links.py` validator enforces it. On-call reading "a rollback fired, now what?" without a runbook is the symptom this guards against.
- **Safety rollbacks have no grace period.** Quality rollbacks have a 24-hour grace to prevent oscillation; safety breaches fire immediately. If you find yourself typing `grace_period: 24h` on a jailbreak-rate trigger, read chapter 02 again.
- **The waiver mechanism keeps the policy honest.** Every real release-gate policy will have edge cases; a policy that forbids waivers forces teams to break the policy instead of filing one. The waiver has signers, an expiry, and a follow-up — those three keep it from becoming "ship without a gate."
- **Bootstrap state is honest, not aspirational.** If the canary machinery is not in place yet, `enforcement: stub_logged` is correct and the binding doc names the hand-off date. The chapter 03 delegation contract with `ai-infra-mlops` is what unblocks the hand-off.
- **The emergency path is counted.** `emergency_release` is a named shape with a count cap per quarter. If a family runs more than two emergency releases per quarter, the policy is wrong (either the gates are too tight or the emergency bar is too low); the quarterly refresh addresses it.
- **The dry-run simulator is not optional.** Shipping a policy without exercising it with a fixture is how a typo in `rollback_criteria.yaml` surfaces as a 2am page. The two fixtures (passing + failing) catch most authoring bugs before CI runs the first release.

## Acceptance criteria

You are done when:

- `policy.yaml` validates against `schema/policy.schema.json` and every validator in `validate/*` passes. Every threshold has a provenance block; every regulated surface has a governance-peer signer; every change class in the map is defined in `change_classes.md`.
- `rollback_criteria.yaml` has one entry per post-ship threshold; every entry references a readable runbook; `validate_runbook_links.py` passes.
- Every surface in the family has a release runbook and (where applicable) a rollback runbook; the waiver-process runbook exists.
- The CI job runs the validators on every PR; a deliberate break (remove a threshold's provenance; introduce an unsigned role; delete a referenced runbook) causes the CI to fail.
- `simulate/fake_release.py` produces the expected per-gate verdicts for both fixtures: `passing_release.yaml` passes offline, generates the ramp plan, arms the rollback triggers; `failing_release.yaml` fails offline on a specific threshold with the provenance citation and does not advance.
- At least one surface's binding to the mod-106 CI, the mod-107 canary / ramp machinery, and the mod-107 post-ship rollback is live (or has a stub + hand-off date documented in the binding doc). "The policy says X; the machinery enforces X" is traceable for one surface.
- `history/2026-Q4-initial.md` documents the authoring choices and the next refresh calendar entry; the next refresh is on the team calendar (shared cal, project tracker, or similar) with the eval program owner as owner.
- The `README.md` is self-standing — a new on-call engineer with no prior context can open the README and find the gate map, the escalation contacts, the rollback runbook pointers, and the refresh cadence without asking anyone.

## Stretch goals

- **Two-family policy composition.** Stand up a second family (even skeletally) and show how the two policies share schemas and runbook templates while differing per-family on surfaces and thresholds. The composition exercises the "one policy per family" discipline against a real second case.
- **Threshold-provenance crawler.** A script that walks every `provenance.source` reference and verifies the source document exists / is reachable. Catches drift when a product doc is renamed or a harm-model refresh moves.
- **Policy-diff review.** On a quarterly refresh, generate a diff of `policy.yaml` (and `rollback_criteria.yaml`) against the previous version with human-readable commentary on each threshold change. The diff is what the quarterly review walks through.
- **Rollback-trigger firing simulator.** Given a hypothetical mod-107 metric timeseries, simulate which rollback triggers would fire, when, and whether the grace-period logic would hold or escalate. Useful for calibrating thresholds without waiting for a real incident.
- **Signer rotation.** Build a signer-rotation shim — when a signer leaves the role, the policy requires re-signature by the successor within an agreed window, and the validator fails until the re-signature lands. Keeps the policy signed-by-the-current-role rather than signed-by-whoever-happened-to-be-there.
- **Audit-walkthrough generator.** From `policy.yaml` + `rollback_criteria.yaml` + the runbook set, generate a one-page audit-walkthrough brief per surface. The brief is what an internal auditor or a procurement reviewer walks through; see chapter 07 for the framework-mapping shape it composes with.
- **Policy signing with Sigstore / DSSE.** Replace the signer dates with cryptographic signatures. Every policy refresh is a signed attestation; the validator verifies signatures rather than name-and-date strings.

## What this exercise does *not* cover

You are not authoring the delegation contracts (exercise 02); the build-vs-buy platform matrix (exercise 03); the investment-ledger infrastructure (exercise 04); the card-slice generator (exercise 05); or the framework mapping (chapter 07 is covered in the labs rather than a dedicated exercise). You are shipping the *release-gate architecture for one family* — the policy document the four downstream exercises' artefacts attach to and compose over.
