# exercise-04: Incident To Regression Fixture Walkthrough

**Estimated effort:** 2 hours

## Objective

Walk one real (or realistic) production incident on a surface in your family's scope end-to-end through the chapter 05 investment loop — attend the retrospective (or stand in for it), open an **investment ledger** row, author the three permanent artefacts (regression fixture, online alert, runbook diff), verify the fixture catches the failure against pre-fix and post-fix code, land the ledger row through *planned → in-progress → landed → verified*, and bind any peer-contract follow-ups to the chapter 03 contracts.

By the end of the exercise one incident has produced durable eval-program coverage: the offline gate now blocks a candidate with the same failure mode, the online loop pages the on-call if the same mode shows up in production, the runbook captures what the incident revealed, and the ledger row is the institutional memory future on-calls search when a similar incident recurs. The chapter 01 symptom "the same regression class recurs quarterly" has one less failure mode on its list.

## Prerequisites

- Chapters 01, 02, 03, and 05 of this module.
- Exercise 01 — the release-gate architecture is the surface the fixture lands into (via the offline gate's eval-set reference) and the alert lands onto (via the post-ship gate's `rollback_criteria.yaml`).
- Exercise 02 — any peer-contract follow-up (missing adversary persona; missing capability-profile refresh) binds to the chapter 03 contracts.
- One **in-scope incident** on a surface the eval program gates. Preferred: a real P0 / P1 from the last 90 days on a surface in your family. Acceptable: a realistic hypothetical you construct from a documented class of failure (OWASP LLM Top-10, a published incident post-mortem, a mod-108 safety finding) — the mechanics are the same.
- Mod-106 (CI-gated eval) at the replay-bundle stage — the fixture lands in the frozen eval set the offline gate scores against.
- Mod-107 (online eval) at the alert-config stage — the alert lands in the online-loop configuration. If mod-107 is still being stood up, author the alert config in the shape the online loop will consume and stub-log the firing path, noting the bootstrap state.
- Mod-108 (safety) if the incident is safety-shaped — the fixture inherits the safety-suite shape from mod-108 chapter 03.
- Mod-110 (platform slice) tables (`eval_sets`, `eval_runs`, `eval_results`); or scaled-down substitutes.
- Access to the incident's triage logs, the on-call rotation, and any post-mortem notes. If no formal post-mortem exists, you will author the equivalent — the ledger row structure doubles as the retrospective shape.
- Python 3.11+; `pydantic` or `jsonschema` for the ledger validator; the test harness used by your mod-106 replay bundle.

## Set-up

1. Create the investment-ledger directory (or extend if a ledger already exists):

   ```
   programs/investment/
   ├── ledger.yaml                          # the central ledger
   ├── incidents/
   │   └── <INC-id>/
   │       ├── post-mortem.md               # authored during the retrospective (or hereafter)
   │       ├── trace-exports/               # sanitised trace / span exports from the incident window
   │       │   └── .gitkeep
   │       └── root-cause-notes.md
   ├── fixtures/
   │   └── <INC-id>/
   │       ├── fixture_cases.yaml           # the 3 – 10 failure cases + paired controls
   │       ├── fixture_rubric.yaml          # how each case is scored (pass / fail criteria)
   │       └── verification_report.md       # pre-fix fail / post-fix pass evidence
   ├── alerts/
   │   └── <INC-id>/
   │       ├── alert_config.yaml            # mod-107 alert definition
   │       └── runbook_stanza.md            # paired runbook stanza
   ├── runbook_diffs/
   │   └── <INC-id>/
   │       └── diff.md                      # PR diff against the surface's runbook
   ├── taxonomy/
   │   └── root_cause_classes.yaml          # queryable class taxonomy per family
   ├── schema/
   │   ├── ledger.schema.json
   │   ├── fixture_cases.schema.json
   │   └── alert.schema.json
   ├── validate/
   │   ├── validate_ledger.py
   │   ├── validate_fixture_coverage.py     # pre-fix fails, post-fix passes, controls pass
   │   ├── validate_alert.py                # metric, cohort, threshold, action, runbook, grace
   │   └── validate_contract_bindings.py    # ledger row contract bindings resolve to real contracts
   ├── review/
   │   ├── monthly_review_template.md
   │   └── summary_bot.py                   # produces the monthly channel summary
   ├── history/
   │   └── 2026-Q4-initial.md
   └── README.md
   ```

2. Open the ledger. Fill `ledger.yaml` with one row for the chosen incident, in the chapter 05 shape:

   ```yaml
   investment_ledger:
     - incident_id: <INC-YYYY-MMDD-slug>
       incident_ref: incidents/<INC-id>/post-mortem.md
       date_opened: 2026-XX-XX
       severity: <P0|P1>
       surface: <surface>
       family: <family>
       incident_commander: <name or placeholder>
       triage_summary: >
         <2–4 sentences — what broke, how it was detected, what the user impact was,
          how it was mitigated at the time>
       root_cause_class: <existing-class-slug or new class authored this cycle>

       fixture:
         status: planned
         owner: <eval engineer embedded with surface>
         deadline: <date, 2–4 weeks out>
         pr: <PR link or TBD>
         landed_at: null
         notes: >
           <what the fixture will contain — number of cases, control shape,
            eval-set version it will land in>

       alert:
         status: planned
         owner: <on-call lead>
         deadline: <date>
         pr: TBD
         landed_at: null
         alert_config: >
           <metric, cohort, threshold, window, action, runbook ref>

       runbook:
         status: planned
         owner: <incident commander>
         deadline: <date>
         pr: TBD
         landed_at: null
         runbook_diff: >
           <what sections of the runbook will change — detection, ramp-down,
            notification templates, escalation contacts>

       ledger_entry_status: partial
       contract_bindings: []        # populate if the retrospective surfaced a peer gap
   ```

3. Author the post-mortem in `incidents/<INC-id>/post-mortem.md` using whatever template your org already has (or the chapter 05 shape if there is none). The post-mortem includes the timeline, the detection path, the root cause, the mitigation, the user impact, and the action items. The eval-program action items are the three artefacts — fixture, alert, runbook diff — tracked in the ledger row.

4. Sanitise and export the incident evidence into `incidents/<INC-id>/trace-exports/` — the exact prompts / inputs / retrieval contexts / tool-call sequences that produced the failure, with PII removed and user ids replaced by stable hashes. These exports are the raw material the fixture is built from; they are *not* the fixture itself.

5. Author `taxonomy/root_cause_classes.yaml` (or extend the existing one) with the root-cause taxonomy per family (chapter 05's example classes: `guardrail_bypass_*`, `retrieval_*`, `judge_*`, `cost_regression_*`, `latency_regression_*`, `cohort_regression_*`). Reuse an existing class for this incident if one fits; add a new class with a one-paragraph definition if none does. The taxonomy's value is the join key — in six months, a search for `root_cause_class: guardrail_bypass_encoded_payload` should surface every prior incident of this shape and their fixtures.

## Requirements

Produce a PR against your eval-program repo that adds:

1. **One ledger row in `ledger.yaml`** — fully populated with the chapter 05 shape; `incident_id`, `root_cause_class`, three-artefact status blocks, peer `contract_bindings` if applicable, `ledger_entry_status` transitioning from `partial` to `complete` across the exercise's work.
2. **`incidents/<INC-id>/post-mortem.md`** — the retrospective doc; timeline + detection + root cause + mitigation + user impact + eval-program action items. If you are standing in for the real retrospective (because the incident predates the exercise), note that explicitly and reconstruct from the triage logs.
3. **`incidents/<INC-id>/trace-exports/`** — sanitised trace / span exports; PII removed; user ids hashed; multi-turn sequences preserved. Document the sanitisation steps in the directory's README (even a short one).
4. **`fixtures/<INC-id>/fixture_cases.yaml`** — 3 – 10 failure cases (variants exercising the same failure mode from different angles) plus 1 – 2 paired controls per case ("same shape but should pass"). Each case: `case_id`, `input`, `expected_failure_signal` (what the pre-fix code outputs that is wrong), `expected_pass_signal` (what the post-fix code outputs that is right), `tags` (including `root_cause_class`, `incident_ref`). Deterministic; small enough to run in seconds.
5. **`fixtures/<INC-id>/fixture_rubric.yaml`** — the rubric or programmatic check that scores each case; references an existing rubric where possible; cites the mod-106 replay-bundle version the fixture lands in.
6. **`fixtures/<INC-id>/verification_report.md`** — the pre-fix / post-fix verification evidence:
   - Pre-fix run: every failure case fails; every control passes (or fails deliberately as noted).
   - Post-fix run: every failure case passes; every control still passes (controls catch over-blocking fixes).
   - The report cites the commit SHAs of the pre-fix and post-fix code and the eval-set version the fixture lands in.
7. **`alerts/<INC-id>/alert_config.yaml`** — the mod-107 alert definition. Required fields (chapter 05):
   - `metric` (named mod-107 metric or an identifiable derivation)
   - `cohort` (scope — the request class or user-cohort that would exhibit the failure)
   - `threshold` (with provenance — baseline + Δ calibrated to the incident's magnitude)
   - `window` (detection latency vs. false-positive trade-off)
   - `action` (`page` + `flag_flip` / `router_tier_restrict` / `ramp_down` / enqueue investment review)
   - `runbook` (path to the paired runbook stanza)
   - `grace_period` (`none` for safety, typically `24h_since_previous_rollback` for quality)
8. **`alerts/<INC-id>/runbook_stanza.md`** — the paired runbook stanza referenced by the alert; what the on-call does when the alert fires; escalation contacts; the pointer back to the ledger row.
9. **`runbook_diffs/<INC-id>/diff.md`** — the Markdown PR against the surface's runbook (the one from exercise 01) with:
   - Response steps that were not in the runbook before (detection path, mitigation commands, query recipes)
   - Escalation contacts that were unclear during the incident
   - Ramp-down / rollback procedure gaps revealed by the incident
   - Notification templates that were missing (customer comms, internal all-hands)
   The diff is reviewed by the incident commander and the on-call lead; link the merged PR.
10. **`taxonomy/root_cause_classes.yaml`** — extended if this incident needs a new class; existing class reused if one fits. One-paragraph definition per class.
11. **`schema/*.json`** — JSON schemas for `ledger.yaml`, `fixture_cases.yaml`, `alert_config.yaml`.
12. **`validate/validate_ledger.py`** — schema + semantic checks:
    - Every artefact block (`fixture`, `alert`, `runbook`) has `status`, `owner`, `deadline`.
    - `ledger_entry_status: complete` requires all three artefact statuses are `landed` or `verified`.
    - `deadline` is in the future when `status` is `planned` / `in_progress`.
    - `root_cause_class` exists in `taxonomy/root_cause_classes.yaml`.
13. **`validate/validate_fixture_coverage.py`** — runs the fixture against the pre-fix and post-fix commits; asserts the pre-fix run fails every failure case and the post-fix run passes every failure case; asserts the controls pass in both runs.
14. **`validate/validate_alert.py`** — asserts the alert config has every required field; threshold has provenance; runbook path is readable; grace period matches the alert class (safety → `none`).
15. **`validate/validate_contract_bindings.py`** — every `contract_bindings` entry references a real contract in `programs/delegation/contracts/` and names an artefact that exists in that contract.
16. **`review/monthly_review_template.md`** — the standing agenda for the monthly ledger review (chapter 05 § "Monthly review of the ledger"): open-row status; landed-row verification; class-trend analysis; peer-contract follow-ups; ledger hygiene.
17. **`review/summary_bot.py`** — produces the monthly channel summary ("this month: 4 rows opened, 3 rows completed, 1 row overdue and escalated"); dry-run output committed as a fixture.
18. **CI integration** — the four validators run on every PR touching `programs/investment/`; the fixture-coverage validator runs the pre-fix / post-fix fixture evaluation as part of CI (even on a toy harness if the real one is heavy).
19. **`history/2026-Q4-initial.md`** — the authoring note: which incident was walked, what root-cause class it landed under, which peer contract bindings were opened, the first monthly-review calendar entry.
20. **`README.md`** — the directory's self-standing overview; how to open a ledger row; how to author the three artefacts; how to run the monthly review; how to search the taxonomy.

## Starter guidance

- **If you don't have a real incident, construct a realistic one from a published class of failure.** The mechanics are the same. OWASP LLM Top-10 publishes classes; the Anthropic / OpenAI / Google system cards document mitigated attack patterns; your mod-108 safety report has findings that could have been incidents. Pick one that would plausibly ship on your surface and build the artefacts.
- **The ledger row is opened at the retrospective, not after.** Chapter 05 is explicit — the enqueue moment is the retrospective itself. The eval program owner (or embedded eval engineer) attends every in-scope retro and opens the ledger row before the retro closes. For this exercise, if the retro has already happened, re-open the row and backfill honestly.
- **Reuse the root-cause class if one fits.** Every incident becoming a new class is the chapter 05 anti-pattern "every incident is a new class." Reuse makes the ledger queryable; the taxonomy's value compounds over time. New classes are a formal act at the monthly review.
- **Fixtures are 3 – 10 cases, not one.** A single case is fragile — a small change to the model / prompt might make it pass without addressing the underlying failure. Author variants that exercise the same failure mode from different angles (different payloads, different wrappings, different cohorts).
- **Controls prevent over-blocking fixes.** For every failure case, author 1 – 2 "same shape but should pass" cases. The base64-jailbreak example from chapter 05: the fixture includes both the jailbreak base64 cases and legitimate base64 content that must still get through. Without controls, the fix might deny the whole class.
- **Sanitise, then re-sanitise.** Traces contain PII, user ids, internal state, and sometimes credentials. The fixture is NOT a copy-paste of the prod trace. Document the sanitisation steps so the next incident's engineer can repeat them.
- **Verify against pre-fix AND post-fix.** The chapter 05 *verified* status transition requires both. Pre-fix: the fixture catches the failure. Post-fix: the fixture (and the controls) pass. A fixture that only verifies post-fix is not proven to catch the failure if the fix regresses later.
- **Alert thresholds have provenance like gate thresholds.** The alert's threshold cites the incident's magnitude as the calibration source — "the incident showed a jailbreak success rate of 1.2% over 6 h; baseline is 0.3%; set threshold at baseline + 0.5pp over 6h rolling, grace period none." The provenance is what the quarterly alert-calibration review reads.
- **Safety alerts have no grace period.** Any breach fires immediately. The chapter 05 warning holds — do not add a grace period to a jailbreak-rate alert to reduce page frequency. If the alert is paging too often, the threshold or the detection scope is wrong, not the grace period.
- **Peer contract bindings are the mechanism for peer follow-ups.** If the incident's root cause exposes a peer-artefact gap (persona missing from the risk-eng peer's library; capability profile out of date; supply-chain SBOM missing), the ledger row's `contract_bindings` block names the contract (`ai-risk-engineer`), the artefact (`adversary_persona_library`), and the follow-up (`peer notified; next persona refresh adds encoded-payload class`). The chapter 03 contract's escalation shape takes it from there.
- **One row per incident, three artefact statuses — not three rows per artefact.** The chapter 05 anti-pattern "one row per artefact, not per incident" is subtle; it loses the incident shape. One row with three nested status blocks keeps the incident coherent and makes the "complete" transition meaningful.
- **The monthly review is where the ledger stays alive.** For this exercise you only walk one incident, but set up the monthly review calendar entry and the summary bot now. Six months of monthly reviews later, the ledger is the eval program's queryable institutional memory.

## Acceptance criteria

You are done when:

- One ledger row in `ledger.yaml` represents the chosen incident; the row validates against `schema/ledger.schema.json`; the three artefact statuses advance from `planned` to `in_progress` to `landed` (and ideally `verified`) across the exercise's work.
- The post-mortem is authored (even if backfilled); the sanitised trace exports are checked in; the sanitisation steps are documented.
- The fixture has 3 – 10 failure cases + paired controls; `validate_fixture_coverage.py` passes (pre-fix fails the failure cases; post-fix passes them; controls pass in both runs).
- The alert config has every required field; threshold has provenance; the paired runbook stanza exists; `validate_alert.py` passes.
- The runbook diff is a merged PR against the surface's runbook; the diff captures the response-step gaps the incident revealed.
- The root-cause class is in `taxonomy/root_cause_classes.yaml` (either reused existing or newly named with a one-paragraph definition).
- Any peer-contract follow-ups are bound via `contract_bindings` to real contracts in `programs/delegation/`; `validate_contract_bindings.py` passes.
- The monthly review template exists; the first monthly-review calendar entry is set; the summary bot produces a readable dry-run output.
- CI runs all four validators on every PR touching `programs/investment/`; a deliberate break (remove a control, set `ledger_entry_status: complete` without landed artefacts, cite a non-existent contract) fails CI.
- A reader opening the ledger row six months from now can trace the incident → root-cause class → fixture → alert → runbook diff → peer follow-up chain in one sitting; the institutional memory is preserved.

## Stretch goals

- **Backfill-the-ledger pass.** Pick three incidents from the last 12 months (even approximate reconstructions) and author ledger rows + fixtures for each. The exercise's taxonomy becomes queryable, and the "class trend" analysis the monthly review runs has real data.
- **Class-trend dashboard.** A script that groups ledger rows by `root_cause_class`, counts incidents per class per quarter, and flags classes with increasing frequency ("we had three `guardrail_bypass_*` incidents this quarter; the fixture coverage needs expansion"). Feeds the monthly review.
- **Fixture-expansion heuristic.** When a second incident of the same `root_cause_class` is opened, suggest expanding the existing fixture rather than authoring a new one (chapter 05's note). The suggestion names the specific fixture file and the new cases to add.
- **Alert-firing-rate review.** Instrument the alert configs with a firing-rate counter; at each quarterly review, flag alerts that have never fired (possibly stale) and alerts that fire weekly (possibly miscalibrated).
- **Pre-fix / post-fix bisect.** For incidents where the mitigating commit is unclear, use `git bisect` with the fixture as the test to identify the first commit where the fixture flipped from fail to pass. The bisect result is the attribution.
- **Chapter 07 escalation hook.** For incidents with governance implications (regulated surface, EU AI Act Article 73 trigger), auto-open an evidence-bundle request to the governance peer via the chapter 03 contract. The bundle contains the ledger row, the fixture, the alert, and the framework-mapping cross-reference.
- **Public-incident ingest.** Scrape (manually or via a watch job) incidents from public sources (OWASP LLM Top-10 updates, published provider post-mortems, academic red-team papers) and open ledger rows for incidents that would plausibly affect your surface. Preventive coverage without waiting for your own incident.

## What this exercise does *not* cover

You are not authoring the release-gate architecture (exercise 01), the delegation contracts (exercise 02), the build-vs-buy matrix (exercise 03), or the card slice generator (exercise 05). You are shipping the *incident-to-investment loop* for one real (or realistic) incident — the chapter 05 artefact that turns a one-off production failure into permanent eval coverage and queryable institutional memory, and that makes the chapter 01 symptom "the same regression class recurs quarterly" one failure mode shorter.
