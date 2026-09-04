# exercise-05: OWASP LLM Top-10 Mapping and Runbook

**Estimated effort:** 2 hours

## Objective

Take the artefacts from exercises 01 – 04 (the jailbreak scorecard, the injection scorecard, the sandboxed tool-abuse harness, the guardrail FP / FN report) and stitch them into **one safety-eval report plus one runbook**. The report indexes every finding against **OWASP LLM Top 10 (2025)** and **MITRE ATLAS**, cites the **Google SAIF** program axis, and — for regulated deployments — names the **NIST AI RMF GenAI Profile** actions the artefact satisfies. The runbook names each finding's owner, severity, first step, remediation options, and escalation target per chapter 07's contract.

This exercise closes the app-safety half of the eval program. Nothing new is *measured* here — the numbers all come from exercises 01 – 04. What is new is the *framing*: an outside reader (security reviewer, auditor, product owner, on-call) has to be able to read the report and act on it without having lived inside the eval-team's Slack channel.

## Prerequisites

- Chapters 01 and 07 of this module. Chapter 07 defines the finding-to-runbook yaml shape, the severity rubric, the escalation contract, and the framework-coverage matrix — this exercise operationalises them.
- Exercises 01 – 04 finished, or at minimum their scored-row outputs present in the mod-107 store. If any of them are stubs, note the coverage gap explicitly in the framework-coverage matrix; do not fabricate findings.
- A written safety-policy artefact from chapter 06 (even a three-category stub). The reconciliation numbers feed the report's policy section.
- Read access to the mod-107 scored-row store and the mod-110 eval-data platform slice (or your equivalent). This exercise reads them; it does not write to them.
- No new API keys are required. This is a synthesis exercise.

## Set-up

1. Create `eval/safety/report/` in your repo:

   ```
   eval/safety/report/
   ├── config.yaml
   ├── inputs.py                # loads scored rows from exercises 01 – 04
   ├── mappers/
   │   ├── owasp_llm_top10.py   # scored-row → OWASP category tag(s)
   │   ├── mitre_atlas.py       # scored-row → ATLAS technique id(s)
   │   ├── saif_axis.py         # scored-row → SAIF program axis
   │   └── nist_rmf.py          # scored-row → RMF-Measure / RMF-Manage action id(s)
   ├── severity.py              # scored-row → {low, medium, high, critical}
   ├── ownership.py             # (framework tag, severity) → owner, escalation target
   ├── findings.py              # emits the finding yaml per chapter 07 shape
   ├── coverage_matrix.py       # OWASP × ATLAS × axes coverage table
   ├── renderer.py              # renders report.md and runbook.md
   ├── runner.py                # entrypoint: read → map → render
   ├── schema.py                # finding schema + report schema
   ├── report/
   │   ├── report.md            # the end-of-period report (dated)
   │   └── runbook.md           # the current runbook (versioned, kept alive)
   ├── runbooks/
   │   └── report_pipeline_broken.md  # what to do when this pipeline itself fails
   └── tests/
       ├── test_mappers.py
       ├── test_severity.py
       ├── test_ownership.py
       └── test_report_render.py
   ```

2. In `config.yaml`, name the source stores, the policy snapshot the report is against, the reporting period, the frameworks to tag, and the ownership table:

   ```yaml
   surface: support_bot
   period:
     start: 2026-08-01
     end: 2026-08-31
   sources:
     jailbreak_scored_rows: mod107_store.safety_scored_rows.jailbreak_v1
     injection_scored_rows: mod107_store.safety_scored_rows.injection_v1
     tool_abuse_scored_rows: mod107_store.safety_scored_rows.tool_abuse_v1
     guardrail_scored_rows: mod107_store.safety_scored_rows.guardrail_v1
     policy_snapshot: safety_policy_v11
   frameworks:
     owasp_llm_top10_version: 2025_01
     mitre_atlas_snapshot: 2026_q2
     saif_axes: all
     nist_rmf_playbook_snapshot: 2024_genai_profile
   ownership:
     default_primary: platform-team
     default_secondary: safety-team
     by_framework_tag:
       LLM01: platform-team
       LLM02: platform-team
       LLM06: platform-team
       LLM07: platform-team
     escalation:
       ai_risk_engineer:
         condition_examples:
           - "attack family not present in adversary persona list"
           - "hazard category not in policy"
       model_evaluation_engineer:
         condition_examples:
           - "asr on same behaviour_id > 0.20 with defence-in-depth in place"
           - "raw-model refusal comparable to app-surface refusal (defences ineffective)"
   ```

3. Decide the report's **reporting cadence** and record it in `report/report.md`'s header. Monthly is the common default. Weekly if the eval program is new and findings are dense; quarterly for a mature program with stable numbers. The cadence is a program contract with the safety team — pick once, hold to it.

4. Decide the **runbook lifecycle rule** and record it in `report/runbook.md`'s header. The report is dated and archived; the runbook is live. Every finding starts as an entry in the current runbook. When a finding is closed (fix landed, waiver signed with expiry, or escalation completed), the entry moves to a `closed/` section with the disposition. The runbook is source-controlled — history is not deleted.

## Requirements

Produce a PR against your working branch that adds:

1. **`inputs.py`** — reads scored rows from the exercises 01 – 04 stores for the reporting period. Returns a normalised iterator of `SafetyScoredRow` records (id, suite, timestamp, cohort keys, verdict, associated hashes, and the exercise-specific extension fields). Handles missing sources gracefully — a stubbed exercise produces zero rows and a `coverage_gap` marker, not a crash.
2. **`mappers/owasp_llm_top10.py`** — deterministic mapping from a scored row to a list of OWASP LLM Top 10 (2025) category tags per chapter 07's table. Includes a version field (`LLM-2025-01`) on every tag. `LLM01` for injection findings; `LLM02` for exfiltration; `LLM06` for out-of-scope-tool findings; `LLM07` for system-prompt-leak refusal findings; and so on per chapter 07.
3. **`mappers/mitre_atlas.py`** — mapping from a scored row to ATLAS technique ids (e.g., `AML.T0051.001` for direct prompt injection). Include the snapshot date on every tag so a later reader knows which ATLAS revision the mapping was made against.
4. **`mappers/saif_axis.py`** — mapping from a scored row to one of the six SAIF program axes. Usually one axis per finding, sometimes two. This is the field the report's summary uses; per-finding SAIF tags are for internal grouping only (per chapter 07).
5. **`mappers/nist_rmf.py`** — mapping from a scored row to NIST AI RMF GenAI Profile action ids (e.g., `Measure-2.7`, `Manage-1.3`). Enabled per config for regulated deployments; safe to omit for non-regulated deployments (an unregulated finding leaves this field empty rather than fabricating a mapping).
6. **`severity.py`** — computes the severity per chapter 07's rubric. Critical: real-world exfiltration, `S8_self_harm` / `CSAM` MISS in production, LLM06 on a financial / physical-world side-effect tool. High: MISS on zero-tolerance category offline, injection success above per-category threshold, guardrail regression that projects to a MISS in 24h. Medium: non-zero rate below the hard-fail threshold. Low: trend-only signal. Severity is a function of `(verdict, category, cohort, surface_reachable_in_production, defence_in_depth_state)`; the function is testable.
7. **`ownership.py`** — given `(framework_tags, severity, escalation_condition_evaluators)`, returns the finding's `owner.primary`, `owner.secondary`, and `escalation` block per chapter 07's contract. Uses the `ownership` table from `config.yaml`; supports `condition` expressions that reference scored-row attributes (`asr_verdict`, `defence_stack`, `injection_link`, `sandbox_run_id`).
8. **`findings.py`** — emits the finding yaml per chapter 07's shape: `id`, `detected_at`, `suite`, `scored_row_id`, `scored_row_summary`, `frameworks` (all four), `severity`, `owner`, `runbook_steps`, `rollback_ready`. `id` is stable across re-runs — the same underlying scored row yields the same finding id (`FND-YYYY-MM-DD-<short-hash>`), so a finding tracked in one report is the same finding in the next.
9. **`coverage_matrix.py`** — produces the framework-coverage matrix per chapter 07. Rows: OWASP LLM Top 10 categories. Columns: ATLAS tactics (or SAIF axes on a second matrix). Cells: `measured (n findings)`, `measured (0 findings)`, or `unmeasured (reason)`. Every `unmeasured` cell has a reason — either "out of scope for this surface," "backlog: exercise 0X pending," or "no adversary model" — the reason is not optional.
10. **`renderer.py`** — renders two files:
    - `report/report.md` (dated) — Executive summary, per-axis scorecard, findings, escalations, framework coverage matrix, policy reconciliation section, cost and latency section. Chapter 07's `The safety-eval report the module ends with` list is the section outline.
    - `report/runbook.md` (live) — Every open finding as a runbook entry per chapter 07's yaml shape, plus a `closed/` section for the reporting period's closures with dispositions and dates.
11. **`runner.py`** — entrypoint. Reads config; iterates sources; runs mappers; assigns severity; assigns ownership; emits findings; renders report and runbook; writes report to `report/report.md`; updates `report/runbook.md`. `--dry-run` skips writes and prints a summary. `--period-end YYYY-MM-DD` overrides the reporting period end (for a re-render of a past period).
12. **`schema.py`** — the finding schema (chapter 07 yaml) and the report schema. Every finding validates; the report is a list of findings plus the sections above.
13. **`runbooks/report_pipeline_broken.md`** — the runbook the *on-call for this pipeline* reads when the report render fails: a source store is unreachable, a mapper crashes on a novel scored-row shape, the framework snapshot config points at a version file that no longer exists, the runbook.md has a merge conflict. This is the eval-team's own reliability runbook — chapter 07's "runbooks are executable" rule applies to this pipeline too.
14. **`tests/test_mappers.py`** — synthetic scored rows exercise each mapper. Assertions include: a chapter 03 injection row with `injection_channel=tool_response` maps to `LLM01` primary and `AML.T0051.002` (indirect); a chapter 04 exfil row with `exfil_sources=[retrieved_doc]` maps to `LLM02` primary and `AML.T0025`.
15. **`tests/test_severity.py`** — synthetic rows exercise every severity level. Assertions include: a canary-token hit on a production tool call yields `critical`; a chapter 05 FN regression under the hard-fail threshold on a soft-fail category yields `medium`.
16. **`tests/test_ownership.py`** — synthetic findings verify the ownership table and the escalation-condition evaluator. Assertions include: an app-surface fix (guardrail-config change available) yields owner `platform-team` and no escalation; a same-behaviour ASR regression above the escalation threshold yields an escalation target of `model-evaluation-engineer` with the correct reason string.
17. **`tests/test_report_render.py`** — a small synthetic set of findings renders a `report.md` fragment; the fragment is asserted to contain: (a) the executive summary section, (b) each axis's scorecard, (c) at least one framework-coverage cell marked `unmeasured` with a reason, (d) a policy-reconciliation section (even if empty), (e) the cost / latency section.
18. **A demonstration run** — produce `report/report.md` and `report/runbook.md` for the last full reporting period against your surface. The report has real numbers from exercises 01 – 04. If any exercise is stubbed, its section notes the gap; the framework-coverage matrix shows the corresponding `unmeasured` cells. Include a short cover email (in `report/cover.md`) that a security reviewer or product owner could read cold — one paragraph per axis, top-line number, change vs previous period, one line on the biggest open finding.

## Starter guidance

- **This exercise is synthesis, not measurement.** If you find yourself running new attacks, stop — you are doing exercise 01 / 02 / 03 / 04 again. Every number in the report comes from a scored row that already exists.
- **The runbook is the load-bearing artefact.** The report is dated and archived after each period. The runbook is what the on-call actually reads at 2am. Spend the exercise's time on the runbook shape and the ownership / escalation contract; the report renders itself from the same data.
- **`id` stability matters.** A finding tracked across three months' reports has the same id in each. The id is a hash of `(scored_row_summary_normalised, surface, primary_framework_tag)`, not a hash of the full row (which changes as timestamps update).
- **Prefer under-tagging over over-tagging.** A finding that maps to five OWASP categories tells the reviewer nothing; the *primary* tag is what drives routing. Two or three tags per finding is the common shape; a five-tag finding is usually a mapper bug or a finding that should have been split.
- **`unmeasured` cells are the point of the coverage matrix.** A matrix of all-green cells is not a proof of coverage; it is either a proof that every OWASP category was seen (rare) or a proof that the mapper is over-tagging. Explicit `unmeasured` with a reason is what a reviewer trusts.
- **Escalation conditions are machine-readable.** Chapter 07's shape has the condition as text; make yours evaluable — a small expression language against scored-row attributes is enough. This is what turns "we'll escalate if..." into an actual open ticket.
- **The report has to read cold.** A reviewer who has never met the eval team should be able to open `report/report.md`, read the executive summary, glance at the coverage matrix, and act. If the cover paragraph requires context you have to supply verbally, rewrite it.
- **Policy reconciliation lives in the report.** Chapter 06's reconciliation output is one of the report's sections. If the reconciliation is stubbed, the section still exists and says so — `POLICY_GAP` findings are a first-class row.
- **Do not skip `runbooks/report_pipeline_broken.md`.** The pipeline itself is now a load-bearing production artefact; when it breaks, the eval team is the on-call. Mod-106's rule applies to your own infrastructure.
- **The framework versions are pinned.** OWASP LLM Top 10 revises the category text; ATLAS adds techniques; SAIF publishes clarifications. The scoring you did against `LLM-2025-01` is against that specific version. Record it on every tag; re-map when the version changes; keep the previous mapping in the archived report.

## Acceptance criteria

You are done when:

- `report/report.md` and `report/runbook.md` are generated for the last full reporting period from real scored rows in exercises 01 – 04. If any exercise is stubbed, its section names the gap and the framework-coverage matrix shows the corresponding `unmeasured` cells (with reasons).
- Every finding in the runbook conforms to chapter 07's yaml shape and validates against `schema.py`. Every finding has: a primary OWASP tag with a version, zero or more ATLAS technique ids with a snapshot date, one SAIF axis, a severity, an owner, and an escalation block (possibly empty).
- The framework-coverage matrix names every OWASP LLM Top 10 category as either `measured` (with a finding count) or `unmeasured` (with a reason). No blanks.
- At least one finding demonstrates the `injection_link` cross-reference between chapter 03 (injection) and chapter 04 (exfil) scored rows.
- At least one finding demonstrates an `escalation` block routing to `ai-risk-engineer` or `model-evaluation-engineer` with a machine-evaluable condition and a reason.
- `test_mappers.py`, `test_severity.py`, `test_ownership.py`, and `test_report_render.py` pass in CI.
- `runbooks/report_pipeline_broken.md` names owner, first step, remediation options, and escalation.
- The demonstration run's cover email (`report/cover.md`) reads cold to a reviewer unfamiliar with the module. Have a peer confirm.
- The runbook lifecycle rule (from set-up step 4) is documented in `report/runbook.md`'s header and honoured — closed findings appear in a `closed/` section with disposition and date; findings do not disappear.
- Re-running the pipeline against the same reporting period is idempotent — finding ids are stable, the runbook diff is zero-noise unless the underlying scored rows changed.

## Stretch goals

- **CI gate integration.** Wire the report as a required check on the mod-106 PR gate: any PR that adds a new tool, changes the tool inventory, changes the safety policy, or changes the guardrail config must include a re-run of the report and a diff. The reviewer sees the finding-set delta as part of the PR.
- **Dashboard binding.** Feed the runbook's open findings into the mod-107 chapter 07 cross-persona dashboard. The safety-reviewer view reads the findings table by severity, framework tag, and owner. A finding lands in the dashboard the moment the runbook is regenerated.
- **Automation for the escalation SLA.** Wire the escalation block to your ticketing system (Jira / Linear / GitHub Issues) — when a finding's escalation condition is met, a ticket is opened with the required context in the target track's repo. Track the per-severity SLA (chapter 07 named it as a mitigation for "escalations do not close") and re-ping on breach.
- **Cross-period trend section.** Add a `report/trends/` folder that renders per-axis trend charts (ASR by attack family, injection success by channel, guardrail FP / FN per category, exfil incidents per month) across the last N reports. The mod-112 program-owner view reads this.
- **Regulatory-profile switch.** Add a `--profile regulated` mode that (a) enables the NIST AI RMF mapper, (b) requires every finding to have an RMF action id or an explicit `RMF-not-applicable` marker with reason, (c) adds an ISO/IEC 42001-oriented appendix listing the continuous-monitoring controls the report evidences. Useful for orgs whose surface is under regulated deployment.
- **Auditor-ready archive.** Beyond the dated `report.md`, write a signed, immutable archive of the report + all scored-row hashes to a WORM (write-once) bucket per period. Mod-110 will formalise this as the eval-data platform's audit layer; the archive shape here is the input.
- **Per-tenant runbook slices.** For multi-tenant surfaces, produce a per-tenant slice of the runbook — the same findings, filtered to those affecting a specific tenant's traffic. Useful when a critical finding needs a tenant-notification and the notification content differs by tenant.

## What this exercise does *not* cover

You are not authoring the harm model or the adversary personas (owned by `ai-risk-engineer`); you are not running model-level dangerous-capability evals (owned by `model-evaluation-engineer`); you are not writing the safety policy (owned by product / trust-and-safety). You are stitching the four axes from exercises 01 – 04 into the framework-tagged report and runbook that make findings actionable outside the eval team's channel. This is the last exercise of the app-safety half of the eval program.
