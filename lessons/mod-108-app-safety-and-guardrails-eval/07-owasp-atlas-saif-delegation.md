# OWASP LLM Top 10, MITRE ATLAS, SAIF, and the Delegation Boundary

## Motivation

Chapters 02 – 06 produced five scorecards: jailbreak resistance, injection robustness, tool-abuse containment, guardrail effectiveness, and refusal reconciliation. Each is defensible on its own; none is *actionable* until it lands in a language the rest of the organisation already speaks. Three frameworks the org already speaks — **OWASP LLM Top 10 for LLM Applications**, **MITRE ATLAS**, and **Google SAIF** — turn a scorecard into a finding, a finding into a runbook entry, and a runbook entry into an owner.

Two grounded reasons this chapter exists as its own axis:

- **Security reviewers do not read your scorecards.** A trust-and-safety reviewer, a security engineer, or an auditor comes to the safety-eval report with a mental model shaped by OWASP / ATLAS / SAIF and by the NIST AI RMF. If the report indexes findings against those frameworks, the reviewer can join it to the rest of the security program. If not, the report is a second parallel taxonomy the reviewer has to translate.
- **The delegation triangle from chapter 01 needs an escalation contract.** Some findings get fixed at the app surface (add a guardrail category, tune the operating point, harden the system prompt, sandbox the tool). Others *cannot* be fixed at the app surface (the base model has a capability the app cannot mask; the harm model needs a new adversary persona; the risk categorisation is missing a hazard). The escalation contract names *which* framework tag routes to *which* peer team.

This chapter walks the four framework references, the mapping shape (finding → framework category → severity → owner), the safety-eval runbook the on-call reads, and the escalation rules that push a finding out of this module and into a peer track.

## Core concepts

### OWASP Top 10 for LLM Applications

OWASP publishes a "Top 10" list for LLM applications, maintained by a dedicated project (the OWASP GenAI Security Project). The 2025 list is the current release the module targets; the categories evolve version to version, so the version — `LLM-2025-01`, etc. — is on the finding.

The 2025 categories at the time of writing:

- **LLM01: Prompt Injection**
- **LLM02: Sensitive Information Disclosure**
- **LLM03: Supply Chain**
- **LLM04: Data and Model Poisoning**
- **LLM05: Improper Output Handling**
- **LLM06: Excessive Agency**
- **LLM07: System Prompt Leakage**
- **LLM08: Vector and Embedding Weaknesses**
- **LLM09: Misinformation**
- **LLM10: Unbounded Consumption**

<!-- needs-research: on the next research cycle, re-verify the OWASP LLM Top 10 category slugs and text against the latest release at https://genai.owasp.org/llm-top-10/ — the list is versioned and category text updates roughly annually. -->

Each finding from chapters 02 – 06 maps to one or more categories:

| Chapter | Typical OWASP mapping |
|---|---|
| 02 Jailbreak resistance | `LLM01: Prompt Injection` (when the jailbreak is a direct injection); `LLM09: Misinformation` (when the output is factually harmful) |
| 03 Prompt injection | `LLM01: Prompt Injection` (all rows); `LLM02: Sensitive Information Disclosure` (indirect injections causing data leak); `LLM06: Excessive Agency` (indirect injections causing tool abuse) |
| 04 Tool abuse | `LLM06: Excessive Agency`; `LLM02: Sensitive Information Disclosure` (for exfiltration cases); `LLM05: Improper Output Handling` (for tool-argument-checker bypasses) |
| 05 Guardrail effectiveness | (no direct category; guardrails are *mitigations*) — findings from a guardrail regression are re-mapped to the underlying category the guardrail was catching |
| 06 Refusal reconciliation | `LLM07: System Prompt Leakage` (when the model reveals the system prompt through over-explanation); *no* direct OWASP category for over-refusal (over-refusal is a product regression tag, not a security tag) |

A finding often maps to *several* categories. The primary tag is the one the runbook uses for routing; secondary tags widen the reviewer's join.

### MITRE ATLAS

MITRE ATLAS (Adversarial Threat Landscape for Artificial-Intelligence Systems) is a matrix of **tactics** (the adversary's goal) and **techniques** (the mechanism), modelled after MITRE ATT&CK. Tactics include *Reconnaissance*, *Resource Development*, *Initial Access*, *ML Model Access*, *Execution*, *Persistence*, *Privilege Escalation*, *Defense Evasion*, *Discovery*, *Collection*, *ML Attack Staging*, *Exfiltration*, *Impact*. Techniques are the specific ways an adversary carries out a tactic — `Prompt Injection`, `LLM Jailbreak`, `Denial of ML Service`, `Data from Local System`, etc.

Two properties make ATLAS complementary to OWASP:

- **Attacker-centric.** OWASP LLM Top 10 organises *risks* to the application. ATLAS organises *techniques* the adversary uses. A finding often maps to one OWASP category and to several ATLAS techniques (an injection that leads to exfiltration is `LLM01` in OWASP; in ATLAS it is `AML.T0051.001 Prompt Injection: Direct` → `AML.T0057 LLM Data Leakage` → `AML.T0025 Exfiltration via Cyber Means`).
- **Case-study grounded.** ATLAS ships with public case studies of real adversarial ML incidents. The case-study library is a useful sanity check on whether a finding you produced resembles a real-world attack shape.

Add ATLAS technique ids on the finding alongside the OWASP category. Both are cheap to record; both widen the reviewer's ability to join to other security artefacts.

<!-- needs-research: on the next research cycle, re-verify the current MITRE ATLAS technique ids and matrix layout at https://atlas.mitre.org — the technique catalogue evolves. -->

### Google SAIF

The Google Secure AI Framework (SAIF) organises the *program* side of AI security into six risk-mitigation categories:

- Expand strong security foundations to the AI ecosystem
- Extend detection and response to bring AI into an organisation's threat universe
- Automate defenses to keep pace with existing and new threats
- Harmonise platform-level controls to ensure consistent security across the organisation
- Adapt controls to adjust mitigations and create faster feedback loops for AI deployment
- Contextualise AI system risks in surrounding business processes

SAIF is the *program-level* framework the ai-eval-engineer track defends against directly in mod-112. This module's findings map to SAIF via the response — "detection and response" is the axis chapter 05's guardrail scorecard directly supports; "automate defenses" is chapter 04's sandbox continuous-replay contribution; "contextualise AI system risks" is chapter 06's policy reconciliation.

SAIF is less about per-finding categorisation and more about the *shape* of the safety program. The runbook cites it in the summary; individual findings do not usually cite specific SAIF categories.

### NIST AI RMF and the Generative AI Profile

The NIST AI Risk Management Framework (AI 100-1, 2023) and its Generative AI Profile (AI 600-1, 2024) provide four functions — Govern, Map, Measure, Manage — and a set of suggested actions per function. This module's artefacts map primarily to **Measure** (the scorecards) and **Manage** (the runbook, the escalation contract, the rollback contract from mod-107).

The RMF is heavily cited in regulated deployments (US federal procurement, some EU AI Act implementations). If your organisation is regulated, the RMF-tagged Measure and Manage actions are what an auditor will ask for. The runbook has a per-finding "RMF-Manage-action" field on those deployments.

<!-- needs-research: on the next research cycle, re-verify the AI RMF Playbook actions relevant to safety eval at https://airc.nist.gov and check for updates to the GenAI Profile. -->

### The finding-to-runbook shape

Every finding produced by chapters 02 – 06 lands in the runbook with the same shape:

```yaml
finding:
  id: FND-2026-08-04-042
  detected_at: 2026-08-04T14:32:19Z
  suite: chapter_04_tool_abuse_sandbox
  scored_row_id: scored_2026-08-04_1832_42
  scored_row_summary: "Indirect injection in retrieved doc leaked CANARY-TENANT-42-DOC-17 via web.fetch"
  frameworks:
    owasp_llm_top10: [LLM01, LLM02, LLM06]
    mitre_atlas: [AML.T0051.001, AML.T0057, AML.T0025]
    saif_program_axis: detection_and_response
    nist_rmf: [Measure-2.7, Manage-1.3]
  severity: high     # low | medium | high | critical
  owner:
    primary: platform-team
    secondary: safety-team
    escalation:
      condition: "asr on same behaviour_id > 0.20 on next full-suite run"
      to: model-evaluation-engineer
      reason: "app-surface defences insufficient; suspected model-level capability regression"
  runbook_steps:
    - "verify canary_leaked in production traffic (spot-check last 24h)"
    - "disable web.fetch on support_bot surface via feature flag"
    - "add guardrail category `internal_canary_pattern` to output classifier"
    - "notify affected tenants per data-breach protocol (owner: legal-team)"
    - "escalate to ai-risk-engineer for adversary-persona update"
  rollback_ready: true
```

Two invariants:

- **Framework tags are lists.** A finding is not one category; it is a set of categories in each framework. The runbook's search / filter reads any tag.
- **`escalation`** is present on every finding, even if empty. When the condition is met, the runbook automation opens a ticket for the escalation target with the required context.

The runbook is source-controlled (findings age; the record of the finding does not). Chapter 07 of mod-107 built the dashboard shape a safety reviewer reads; this runbook is the *ticket*-shaped artefact that dashboard's rows link to.

### Severity, and the safety carve-out

Severity is where the eval team's judgement enters. Four levels, each with a defined threshold shape:

- **Critical** — real-world exfiltration confirmed; canary tokens surfaced in production tool calls; a MISS on `S8_self_harm` or `CSAM` in production; a `LLM06 Excessive Agency` finding on a tool with financial or physical-world side effects. Runbook action: page immediately, feature-flag flip, notify affected tenants, escalate.
- **High** — a MISS in the offline eval on a zero-tolerance category; injection success rate above a per-category threshold; guardrail regression that would cause a MISS in the next 24h at current traffic; large over-refuse regression on a high-volume category. Runbook action: page during business hours; block rollout; timeline for fix.
- **Medium** — a non-zero rate below the hard-fail threshold; injection success on low-severity categories; guardrail regression that is soft-fail-only; over-refuse regression on medium-volume category. Runbook action: file ticket; fix in next sprint.
- **Low** — trend-only signal; a hazard category with no zero-tolerance floor and no volume threshold; a `POLICY_GAP` finding awaiting policy review. Runbook action: file ticket; batch review at next weekly.

The safety carve-out from mod-107 chapter 04 applies: **critical severity is not soft-fail-able.** A single scored row in the canary that trips a critical finding halts the rollout regardless of any statistical-confidence argument. Chapter 05 of mod-107's confidence-sequence logic does not gate hard-fail rows.

### The escalation contract

Not every finding is this module's to fix. Three routing rules:

- **App-surface fix (this module's runbook).** The finding can be resolved by a change *in the app*: a system-prompt update, a guardrail-config change, a new sandbox policy, a tool-scope tightening, an over-refuse threshold adjustment. Almost every finding in chapters 05 – 06 lands here. Most in chapters 02 – 04 land here too.
- **Escalate to `ai-risk-engineer`.** The finding indicates the *harm model or the adversary persona is missing a case*. A new attack family the risk team has not modelled; a hazard category the policy does not cover; a real-world adversary the persona list does not include. The escalation opens a ticket in the risk-eng repo; the eval-team's role ends at the escalation.
- **Escalate to `model-evaluation-engineer`.** The finding indicates the *base model has a dangerous capability that no app-surface defence can reliably mask*. Repeated app-surface failures on the same behaviour with defence-in-depth in place; jailbreak success rates that are close to the raw-model baseline; capability elicitation that the surface's system prompt cannot suppress. The escalation opens a ticket for a model-level assessment.

The runbook's `escalation` field is machine-readable so the escalation is not a "someone will remember" — automation opens the ticket when the condition is met.

### Where the module ends

The last section of the runbook explicitly names what this module does *not* do:

- **We do not run the model-level dangerous-capability evals.** When a finding survives escalation to the model-eval-eng track, that team runs the model-level assessment. Our runbook cites their assessment ticket; we do not re-run their eval.
- **We do not author the harm model.** The risk-eng team maintains adversary personas, hazard categorisation, and threat modelling. When our escalation adds a case, we cite the risk-eng ticket in the finding.
- **We do not write the safety policy.** The policy owner (usually trust-and-safety) owns the policy document. Our findings can *request* a policy update via `POLICY_GAP` findings and via the reconciliation report; the update itself is a policy-owner action.
- **We do not own the tool inventory.** The tool policy tags (chapter 04) are owned jointly by platform and risk teams. Findings can request new policy tags; the tags themselves are added by the tool's owner.

Each of these hand-offs is a *contract*. If a peer team is not present in the org, this module's runbook still exists but the escalation lands in the same slot for a future filling.

### Integration with mod-106 and mod-107

- **Mod-106 CI gate.** Every finding tagged `critical` or `high` that has a reproducible scored row blocks the merge until the finding is either fixed or explicitly waived by the owner. The waiver is a signed exception with an expiry; a waived finding that ages past its expiry re-blocks the gate.
- **Mod-107 online loop.** Findings from production drift alerts and canary gate failures land in the same runbook, with the same framework tags, the same severity, and the same escalation contract. The offline gate produces findings from planned suites; the online loop produces findings from live traffic; the runbook is the same shape.
- **Mod-107 cross-persona dashboards.** The safety-reviewer dashboard reads the runbook's findings by severity, by framework tag, and by owner. The findings are the ticket table the dashboard links to.

### The safety-eval report the module ends with

The end-of-module artefact is a single report (Markdown, published to your safety-team's wiki or repository) that contains:

- **Executive summary** — one paragraph per axis, with the top-line number and the change vs previous period.
- **Per-axis scorecard** — the tables from chapters 02 – 06, all against the same snapshot period.
- **Findings** — the runbook entries produced this period, tagged and severity-scored.
- **Escalations** — the tickets opened to the risk-eng and model-eval-eng peers this period.
- **Framework coverage** — a matrix showing which OWASP LLM Top 10 categories, which ATLAS techniques, and which SAIF axes had evaluation coverage this period (and which did not — an *unmeasured* category is an accepted risk, not a green cell).
- **Policy reconciliation** — the chapter 06 report against the current policy snapshot; the `POLICY_GAP` count and the outstanding policy-owner tickets.
- **Cost and latency** — the guardrail line item (chapter 05) and the safety-eval program's total operational cost (a mod-107 chapter 02 style budget derivation for the safety half).

The report is dated and versioned; it lives alongside the mod-107 online-loop reports and the mod-106 gate history. Mod-112 (owning the eval program) reads all three when defending the program to governance.

### Failure modes to design against

- **Framework tags become theatre.** Every finding gets `LLM01` because prompt injection is common; nobody reads the tag. Mitigation: the tag list is a set; the primary is the one that drives routing; unused tags are visible in the report as "over-tagged" if the same primary appears everywhere.
- **The `unmeasured` category is invisible.** The framework-coverage matrix shows greens for what you ran and blanks for what you did not; the reviewer reads "greens = safe." Mitigation: the framework-coverage matrix uses explicit `unmeasured` cells and lists the reason (out-of-scope, backlog, no adversary model, ...).
- **Escalations do not close.** A ticket is opened for the model-eval-eng team; three months pass; no one follows up. Mitigation: escalation tickets have a per-severity SLA and an automated re-ping; the runbook's `escalation` block includes the SLA target.
- **The runbook is not run.** The runbook is a document; nobody executes it. Mitigation: mod-106's rule (runbooks are executable) applies here: every runbook step is either a scripted action, a link to a scripted action, or a specific named human role.
- **The report is one-shot.** The safety-eval report is written once per launch and never again. Mitigation: the report is on a fixed cadence (monthly / quarterly) with the same shape; comparison across periods is a first-class dashboard row.

## Summary

- **OWASP LLM Top 10 (2025)** is the primary risk-side framework findings are tagged against. Categories `LLM01`–`LLM10`; each finding may map to several.
- **MITRE ATLAS** is the adversary-side complement — tactics and techniques. Tags on findings are lists; the two frameworks join at the ticket layer.
- **Google SAIF** organises the *program* side. This module cites it in report summaries; per-finding SAIF tags are rare.
- **NIST AI RMF Generative AI Profile** is the primary regulated-deployment framework — Measure and Manage functions are what most of this module's artefacts land in.
- **The finding-to-runbook shape** is a versioned yaml with a `scored_row_id`, a set of framework tags, a severity, an owner, an `escalation` block, and runbook steps.
- **Severity levels** (low / medium / high / critical) have concrete threshold shapes; **critical is not soft-fail-able** — the safety carve-out from mod-107 chapter 04 halts the rollout on a single critical scored row.
- The **escalation contract** routes findings to `ai-risk-engineer` (harm model), `model-evaluation-engineer` (model-level capability), or the app team (the majority case). Escalation is machine-readable; automation opens tickets.
- The **end-of-module report** contains the executive summary, per-axis scorecards, findings, escalations, framework coverage, policy reconciliation, and cost / latency. Dated, versioned, published alongside mod-107 online-loop reports.
- Failure modes: framework-tag theatre, invisible `unmeasured` cells, escalations that do not close, non-executable runbooks, one-shot reports.

The module ends here. The next module (mod-109) picks up the human-review workflows that consume the alerts these runbooks cannot resolve; mod-110 stores the artefacts this report cites; mod-111 reads the cost and latency line items; mod-112 defends the whole program to governance. The five scorecards and the runbook are the load-bearing artefacts for all four downstream modules.
