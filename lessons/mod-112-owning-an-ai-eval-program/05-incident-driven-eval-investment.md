# Incident-Driven Eval Investment: Incident → Fixture → Alert → Runbook

## Motivation

Every eval program that runs long enough will ship a regression. The gate policy from chapter 02 will pass a candidate; the canary and ramp gates will look clean; the post-ship rollback triggers will fail to fire because the failure mode was outside the trigger set. A production incident will happen. The question is not whether it happens — it is whether the next occurrence of the same failure is a new incident, or is caught by a permanent fixture the eval program authored the first time.

The default state is that incidents *do not* produce permanent fixtures. The retrospective happens; the fix is deployed; the on-call moves on to the next incident. Six months later the same class of failure recurs and the team relearns it. The retrospective's action items become a wiki page nobody reads.

The fix is a **disciplined investment loop** — every P0 / P1 incident in scope produces (a) a permanent regression fixture that catches the same failure at the offline gate, (b) an online alert that catches the failure at the post-ship gate if the fixture misses, and (c) a runbook diff so the response is faster the second time. The loop has a named owner per incident, a documented deadline, and an **investment ledger** the program owner reviews monthly.

The chapter's payoff: after a year of running the loop, the offline gate's fixture set is a coverage map of every failure mode the org has actually encountered; the online alerts are calibrated to what actually breaks in prod; the runbooks are the collected wisdom of the on-call rotation. The program compounds.

Two symptoms show the loop is missing:

- **Retrospective action items live in Google Docs.** They are written after the incident; nobody owns them; six months later "add regression test" is still open. The intent was correct; the mechanism was absent.
- **The same regression class recurs quarterly.** A safety category triggered by a specific payload shape; a factual-drift on a specific model snapshot; a cost-blowup on a specific tool-call pattern. Every time it recurs, the team treats it as new. The loop closes only if you close it.

Both symptoms are process failures. This chapter closes them.

## Core concepts

### The three permanent artefacts every in-scope incident produces

Every P0 / P1 incident in scope produces three permanent artefacts. Each is a specific file in a specific location with a specific owner.

- **Regression fixture.** A test case (or a small set of cases) added to the frozen eval set the offline gate scores against, such that a candidate exhibiting the same failure mode fails the offline gate. Lives in the mod-106 replay bundle for the affected surface. Owner: usually the eval engineer embedded with the surface.
- **Online alert.** A mod-107 alert (a metric threshold on a cohort, a drift signal, a guardrail-trigger rate) that fires when the same failure mode shows up in production. Lives in the mod-107 alert config for the affected surface. Owner: usually the on-call lead for the surface.
- **Runbook diff.** A change to the surface's release / rollback runbook (chapter 02 of this module; mod-106 chapter 03 at the practitioner level) that captures the response steps the incident revealed. Lives in the runbook file for the affected surface. Owner: usually the incident commander for the incident.

Three artefacts, three owners, three deadlines (typically 2 – 4 weeks after the incident-close). If any of the three is not produced, the incident is not closed on the investment ledger — the retrospective is over, but the investment is not.

### The investment ledger

The **investment ledger** is the central artefact of this chapter. One row per P0 / P1 incident in scope, tracking the three artefacts through to completion. The ledger is reviewed by the eval program owner monthly (chapter 01's cadence).

Shape:

```yaml
investment_ledger:
  - incident_id: INC-2026-0817-support-jailbreak
    incident_ref: incidents/INC-2026-0817-support-jailbreak.md
    date_opened: 2026-08-17
    severity: P1
    surface: support_bot
    family: customer_facing_support_agents
    incident_commander: jamie
    triage_summary: >
      Novel jailbreak pattern (base64-encoded suffix + persuasion frame) leaked
      internal tool schema through the support_bot. Guardrail chain missed the
      encoded portion. 12 sessions affected before manual rollback.
    root_cause_class: guardrail_bypass_encoded_payload
    
    fixture:
      status: in_progress            # planned | in_progress | landed | verified
      owner: casey
      deadline: 2026-09-14
      pr: eval-repo#4321
      landed_at: null
      notes: >
        Add 8 base64-encoded jailbreak cases to support-eval-safety-v14;
        include 3 controls that must still pass (non-jailbreak base64 content).
    
    alert:
      status: landed
      owner: pat
      deadline: 2026-09-07
      pr: infra-repo#8829
      landed_at: 2026-09-03
      alert_config: >
        mod-107 alert on guardrail.jailbreak_success_rate for
        base64-encoded input class; threshold pre-shipped baseline + 0.2pp
        over 6h rolling window; escalation to risk_eng + on_call.
    
    runbook:
      status: landed
      owner: jamie
      deadline: 2026-09-07
      pr: eval-repo#4315
      landed_at: 2026-09-02
      runbook_diff: >
        Added § 3.4 to support-bot-jailbreak-rollback.md — detection of
        encoded-payload class, ramp-down procedure, notification list.
    
    ledger_entry_status: partial     # partial | complete | overdue | reopened
    contract_bindings:
      - contract: ai-risk-engineer
        artefact: adversary_persona_library
        note: >
          Persona library did not include encoded-payload jailbreak; peer
          notified 2026-08-19; next persona refresh (2026-Q4) adds
          encoded-payload class.
      - contract: ai-infra-security
        artefact: attack_signal_enrichment
        note: >
          Peer enriched finding with MITRE ATLAS AML.T0051.001 (Direct
          Prompt Injection); enrichment landed 2026-08-21.
```

Six things the ledger row does:

- **Names the incident precisely.** Not "the jailbreak thing"; a specific incident id, a specific reference, a specific severity, a specific surface and family, a specific commander.
- **Names the root-cause class.** Not "the model got confused"; a specific class the fixture and alert are scoped to (`guardrail_bypass_encoded_payload`). Classes are shared across incidents — a second incident with the same class often expands the existing fixture rather than adding a new one.
- **Tracks three artefacts by status.** Planned / in-progress / landed / verified for each; owner, deadline, PR link.
- **Names the peer contract bindings.** If the root cause exposes a peer artefact gap (an adversary persona missing; a supply-chain check missing), the row references the contract and the follow-up. Chapter 03's contracts are how peer follow-ups get accountable owners.
- **Names the ledger-entry status.** The row is *complete* only when all three artefacts land and are verified. Partial rows are visible at the monthly review; overdue rows trigger escalation.
- **Preserves the wisdom.** Six months from now, a similar incident's commander can search the ledger for `root_cause_class: guardrail_bypass_encoded_payload` and read the previous incident's notes, fixture, alert, and runbook diff. The ledger is *the* institutional memory for eval failures.

The ledger is a single file (YAML or a small SQLite / Postgres table); it is committed to the eval-program repo; it is queried by the monthly review script.

### Which incidents are in scope

Not every production incident should hit the ledger. The scope is the set of incidents where an eval-program action *could* have prevented or mitigated the failure.

In-scope incidents (rough shape; per-family policy makes it precise):

- **Any P0 / P1 on a surface in the release-gate architecture.** The eval program owns the surface's gates; when the gates fail to prevent the incident, the eval program owns the retrospective's evaluation action items.
- **Any safety event (regardless of severity) on a surface with a mod-108 safety report.** Safety events compound; low-severity events today are high-severity if a novel attack pattern goes viral.
- **Any cost or latency incident whose root cause is in the model / prompt / chain / retriever configuration.** The mod-111 trade-off report is the eval program's evidence base; cost / latency regressions from the config layer belong to the eval program.
- **Any incident with a governance implication.** A shipped surface trips a policy line (an EU AI Act GPAI notification, an internal ethics-review flag). The governance peer owns the governance response; the eval program owns the eval-evidence action items.

Out-of-scope (not "unimportant" — just not the eval program's ledger):

- **Pure infra incidents** (the serving cluster went down; the inference API rate-limited). The `ai-infra-mlops` peer's postmortem process. Chapter 03's contract with that peer names the interface.
- **Pure product incidents** (a UI copy change confused users). Product retrospective.
- **Vendor incidents** (a model provider had an outage). The `ai-infra-mlops` peer's incident + the delegation-contract escalation with the vendor.

The eval-program owner joins the incident's retrospective for in-scope incidents; the ledger row is opened there.

### The retrospective enqueue moment

The moment the ledger row is opened is the retrospective itself. The eval-program owner (or the eval engineer embedded with the surface) attends every in-scope retrospective; the retrospective's meeting facilitator adds an "eval investment" section to the notes; the ledger row is opened before the retrospective closes.

The retrospective enqueue captures:

- **Root-cause class.** A short slug — `guardrail_bypass_encoded_payload`, `retrieval_index_stale`, `cost_regression_tool_call_loop`. Shared with any prior incident of the same class; reuse the slug if there is one.
- **Which of the three artefacts are needed.** Some incidents need all three; some need only a fixture + runbook (the failure was caught at canary — no post-ship alert needed); some need only an alert (the failure is not reproducible in fixture form). The retrospective decides.
- **Owner per artefact, deadline per artefact.** The deadline is not "as soon as possible"; it is a specific date, usually 2 – 4 weeks out.
- **Peer contract bindings.** If the retrospective surfaced a peer-artefact gap, the peer contract is named and the follow-up is scheduled.

The enqueue is the first commitment; the ledger row is opened; the monthly review verifies progress.

### The fixture construction pattern

Fixture construction is the highest-signal artefact of the three because it turns the failure into a permanent test. The pattern per incident:

- **Extract the input.** The exact prompt / input / retrieval-context / tool-call sequence that produced the failure, sanitised of any PII. If the input was multi-turn, capture the sequence.
- **Extract the failure signal.** The specific output (or output pattern) that was wrong. For safety failures, the specific tokens or behaviours that violated policy. For quality failures, the specific rubric score band the response fell into.
- **Author 3 – 10 cases, not one.** A single case is fragile — a small change to the model / prompt might make it pass without addressing the underlying failure. Author variants that exercise the same failure mode from different angles.
- **Author paired controls.** For each failure case, author 1 – 2 "same shape but should pass" cases (the base64 example above — the fixture includes both jailbreak base64 cases and legitimate base64 content that must still get through). Controls prevent the fix from over-blocking.
- **Land the cases in the eval set.** Update the eval-set version (chapter 03 of mod-110); increment the version; note the fixture in the eval-set's change-log.
- **Verify the offline gate catches the failure.** Re-run the offline gate against the pre-fix code; the fixture must fail. Run against the post-fix code; the fixture must pass (and the controls must still pass). This is the *verified* status transition.

The fixture is *not* a copy-paste of the prod trace. Traces contain PII, user identifiers, and often internal state; fixtures are sanitised, deterministic, and small enough to run in seconds. Chapter 03 of mod-108 walked the payload-management contract; the fixture pattern honours it.

### The alert construction pattern

Alerts catch failures the fixture missed — either because the failure only manifests at production scale, only manifests with specific real-world inputs, or emerges over time from drift. Pattern:

- **Pick a mod-107 metric.** From the online-loop metric catalogue — a rubric score, a drift signal, a guardrail-trigger rate, an anomaly detector.
- **Scope the metric to a cohort or a request-class.** Alerts on the whole surface's rubric mean are usually too broad to be actionable; scope to the cohort or the input class that would exhibit the failure.
- **Set a threshold and a window.** The threshold has provenance (baseline + Δ, from the pre-ship baseline, calibrated to the incident's magnitude). The window balances detection latency against false-positive rate.
- **Bind an action.** For safety alerts: page + auto-restrict traffic. For quality alerts: page + hold the ramp if still in progress; enqueue an investment-ledger review if in-production. For cost / latency alerts: page + notify the mod-111 cost report reader.
- **Bind a runbook.** Every alert has a paired runbook stanza; chapter 02 of this module walked the shape.
- **Bind a grace period for false-positive control.** First fire in a rolling window is automatic; second fire in the same window requires human confirm.

Alerts age. A quarterly review of every alert's firing rate keeps calibration — an alert that never fires might be stale (the underlying signal changed and the alert no longer detects); an alert that fires weekly might be miscalibrated (threshold too tight or too broad).

### The runbook diff pattern

Runbook diffs capture what the on-call learned during the incident that the runbook did not already contain. Pattern:

- **Identify the response steps that were not in the runbook.** "We had to check the guardrail's decision logs — that was not in the runbook; add it." "The mod-110 lineage-key query to find affected sessions was written from scratch during the incident — capture the query in the runbook."
- **Identify the escalation contacts that were unclear.** "It was not clear who to page for the risk-eng peer; we escalated to the wrong person for 20 minutes. Update the contact list."
- **Identify the ramp-down / rollback procedure gaps.** "The `flag_flip` command was in the runbook but the flag-name was wrong. Fix." "The rollback took 15 minutes because we did not know about the pre-canned rollback flag. Add the flag to the runbook."
- **Identify the notification templates that were missing.** "The customer communication draft was written under pressure; the template should be pre-canned. Add." "The internal all-hands post was ad-hoc; capture the template."

Runbook diffs are Markdown PRs against the runbook file; they are reviewed by the incident commander and the on-call lead. The PR merges before the ledger row transitions to *landed*.

### Monthly review of the ledger

The ledger is reviewed monthly by the eval program owner. Chapter 01's cadence names the ceremony; the ledger is the artefact.

Review agenda:

- **Every open row.** Status per artefact; any overdue deadlines; any blockers.
- **Every row landed since last review.** Are the fixtures verified? Are the alerts firing appropriately (not too often, not too rarely)? Are the runbook diffs referenced in on-call training?
- **Root-cause-class trends.** Are we seeing the same class repeatedly? If yes, the fixture / alert is missing something; expand rather than close.
- **Peer-contract binding follow-ups.** For every row with a contract binding, did the peer's follow-up land? If not, escalate.
- **Ledger hygiene.** Are there rows opened by past incident retrospectives that have been abandoned? Are there in-scope incidents from the past month with no ledger row? Both indicate a gap in the enqueue process.

The review produces (a) an updated ledger with status transitions, and (b) a short summary posted to the eval-program channel — "this month: 4 rows opened, 3 rows completed, 1 row overdue and escalated." The summary is what makes the program's work visible.

### The delegation to peer contracts

The investment loop and the delegation contracts (chapter 03) are tightly coupled. Two shapes:

- **Incident surfaces a peer-artefact gap.** The incident's root cause is a peer-owned artefact that was missing or stale — a persona missing from the risk-eng peer's persona library; a supply-chain check missing from the infra-security peer's SBOM tracker; a capability profile out of date for the model-eval-eng peer. The ledger row's `contract_bindings` names the peer contract and the follow-up. The peer's next artefact refresh incorporates.
- **Peer-side incident surfaces an eval-program gap.** A peer's incident (say, an infra-security event) has an eval-program implication — the eval program's trace access exceeded expected volume; a judge model in the eval pipeline was found to have a known vulnerability. The eval program opens a ledger row scoped to *its* three-artefact response (fixture / alert / runbook analogue for the eval pipeline itself, when applicable).

Neither shape is "eval owns the whole response" or "peer owns the whole response." The contracts split ownership; the ledger tracks the split.

### The classification discipline

The `root_cause_class` slug matters more than it looks. It is the join key that makes the ledger's institutional memory work.

Classes are shared across incidents; adding a new class is a formal act (typically at the monthly review). A rough taxonomy for the customer-facing app family:

- `guardrail_bypass_*` — subclasses per attack shape (encoded payload, obfuscation, translation, tool-call injection, indirect injection via retrieved content, ...)
- `retrieval_*` — subclasses per retrieval failure (stale index, wrong chunker, wrong embedding, retrieval-quality drift, ...)
- `judge_*` — subclasses per judge failure (drift, over-refusal, cohort bias, ...)
- `cost_regression_*` — subclasses per cost-blowup shape (tool-call loop, prompt inflation, cache miss, ...)
- `latency_regression_*` — subclasses per latency shape (retry storm, tool round-trip, streaming stall, ...)
- `cohort_regression_*` — subclasses per cohort shape (locale, tenant tier, input length, ...)

The taxonomy per family is documented alongside the ledger. New incidents reuse an existing class if possible; new classes are named at the monthly review and added to the doc.

The taxonomy's value: after a year, the ledger is queryable by class. "How many `guardrail_bypass_encoded_payload` incidents this year? Show me all their fixtures." The team's institutional memory becomes searchable.

### Anti-patterns to avoid

**"Action items in a wiki page."** The retrospective ends; the action items land in a wiki page nobody owns; six months later the same regression recurs. Fix: the ledger row is the action-item container; the wiki page is a debugging surface at best.

**"One row per artefact, not per incident."** Splitting the fixture, alert, and runbook diff into three separate ledger rows loses the incident-shape. Fix: one row per incident with three artefact statuses.

**"Fixture without controls."** The fixture catches the failure but the fix over-blocks (denies legitimate inputs of the same shape). Fix: paired controls in every fixture set.

**"Alert without a runbook."** The alert fires; the on-call has no procedure; the response is ad-hoc. Fix: no alert lands without a paired runbook stanza.

**"Ledger set once, never reviewed."** The ledger accumulates rows; nobody reviews progress; overdue rows go unnoticed. Fix: monthly review as ceremony; escalation for overdue rows.

**"Every incident is a new class."** Every incident gets a fresh slug; the ledger becomes un-queryable; no institutional memory. Fix: reuse classes; new classes named at the monthly review.

**"Peer follow-ups without contract binding."** The retrospective says "risk-eng peer will update the persona library"; six months later the persona library is unchanged. Fix: contract-binding column in the ledger; the peer follow-up is tracked in the delegation contract's escalation shape.

### The end-of-chapter posture

By the end of this chapter you can:

- Attend every in-scope incident retrospective as the eval-program owner and open an **investment ledger row** with three-artefact status, owner, deadline, root-cause class, peer contract bindings.
- Author the three permanent artefacts per incident — **regression fixture** (3 – 10 cases + controls, verified against pre-fix and post-fix code), **online alert** (mod-107 metric + threshold + window + action + runbook + grace period), **runbook diff** (Markdown PR with the response steps the incident revealed).
- Run the **monthly ledger review** — status per open row; landed-row verification; class-trend analysis; peer contract follow-ups; ledger hygiene.
- Maintain a **root-cause-class taxonomy** per family that makes the ledger queryable and the institutional memory searchable.
- Feed the loop into the **delegation contracts** (chapter 03) so peer-artefact gaps have accountable follow-ups.

Chapter 06 opens the product-side card slice — the composed artefact the governance peer folds into the org-wide model / system card.

## Summary

- Every P0 / P1 incident in scope produces three permanent artefacts: a **regression fixture** (offline-gate coverage), an **online alert** (post-ship-gate coverage), a **runbook diff** (response-speed capture). Three owners, three deadlines.
- The **investment ledger** is the central artefact — one row per incident, tracking the three artefacts through *planned → in-progress → landed → verified*. Reviewed monthly by the eval program owner.
- **In-scope incidents**: any P0 / P1 on a gated surface; any safety event on a surface with a mod-108 report; any cost / latency incident rooted in configuration; any incident with governance implications. Out-of-scope: pure infra, pure product, pure vendor.
- The **retrospective enqueue moment** is where the ledger row opens — root-cause class, needed artefacts, owner per artefact, deadline per artefact, peer contract bindings.
- **Fixtures**: 3 – 10 cases + paired controls; sanitised; deterministic; verified against pre-fix and post-fix code.
- **Alerts**: mod-107 metric + cohort scope + threshold with provenance + window + action + runbook + grace period; quarterly firing-rate calibration.
- **Runbook diffs**: captured response steps, escalation contacts, ramp-down / rollback procedure, notification templates.
- **Monthly ledger review**: open-row status; landed-row verification; class-trend analysis; peer-contract follow-ups; ledger hygiene.
- **Root-cause-class taxonomy** (`guardrail_bypass_*`, `retrieval_*`, `judge_*`, `cost_regression_*`, `latency_regression_*`, `cohort_regression_*`) makes the ledger queryable and the institutional memory searchable.
- **Anti-patterns**: action items in a wiki page, per-artefact rows losing the incident-shape, fixtures without controls, alerts without runbooks, ledger set once and never reviewed, every incident a new class, peer follow-ups without contract binding.

Chapter 06 opens the product-side card slice — the composed artefact that hands the eval program's evidence to the governance peer with zero rewrites.
