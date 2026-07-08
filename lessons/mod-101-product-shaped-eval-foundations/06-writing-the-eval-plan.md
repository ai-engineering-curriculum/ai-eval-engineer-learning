# Writing the Two-Page Eval Plan

## Motivation

Everything before this chapter is preparation. The output — the thing you ship to stakeholders — is a compact plan that a product manager, a machine-learning engineer, and a governance owner can read, agree with, and sign. If it takes longer than fifteen minutes to read, it will not get signed. If it is missing sections, someone will silently paper over the gap. This chapter gives you a fixed template, the rules for filling each section, and a worked example that ties the meeting-summariser thread from chapters 02–04 together.

## Why two pages

The two-page constraint is not aesthetic. It exists because:

- **PMs read fast, and skim.** A five-page plan is not read; it is filed.
- **ML engineers read for the SLIs and the ship-gates.** Everything else is context.
- **Governance owners read for scope, delegation, and the interface to their framework.** They will not scroll past page two to find it.

Compress by removing background, not by removing decisions. If a section feels too long, ask whether it re-explains what the spec or the standards already say.

## The template

Fill these sections in this order. Every section has a minimum bar (in italics) that must be met before the plan is signable.

### 1. Surface & scope (5–10 lines)

- One sentence naming the surface and its user intent contract.
- One sentence naming the runtime shape (single-shot / chat / RAG / tool-calling agent / streaming).
- Named owners: **PM**, **ML engineer**, **AI eval engineer** (you), **governance owner**. Real names, not roles.
- One sentence naming the model + prompt-chain + retriever + tools + guardrails + decoding config version this plan applies to.
- **Out-of-scope callouts** — other surfaces this plan does *not* cover.

*Minimum bar: a reader who has never seen the product can tell what is being evaluated and what is not.*

### 2. Criteria (fits on the first page)

The chapter 02 decomposition, as a table. One row per criterion:

| # | Family | Criterion | SLI | SLO | Gate | Placement |
|---|---|---|---|---|---|---|
| 1 | Task success | ... | ... | ... | merge / canary / watch | offline / online / both |

At least one row per criteria family (task success, groundedness, safety, cost, latency). If a family is absent, it is called out in section 6 (Deferred / Out of scope).

*Minimum bar: every row is falsifiable and numeric. Every SLO names a window and a population. Every row has a gate class and a placement.*

### 3. Offline regression suite (short)

- Fixture sources (author-crafted / snapshot replay / adapted public benchmark).
- Fixture count (order-of-magnitude, per criterion).
- Which SLIs run on this suite; which SLOs are merge gates versus canary gates.
- The peer / platform interface: which CI runner (`mod-106`), which fixture-management path (`mod-110`), pinned model / prompt / retriever / decoding / seed / fixture hashes.
- Cost budget for a single CI run.

*Minimum bar: someone can rerun this suite offline from the plan alone (with reference to the repo it lives in).*

### 4. Online sampled-trace eval (short)

- Sampling policy (rate, stratification).
- Which SLIs run here; window and reporting cadence.
- Judge or rule stack; per-family scorer.
- Alert / page thresholds and who is on call.
- Judge and infrastructure cost budget.
- Cross-reference to canary / shadow policy (`mod-107`).

*Minimum bar: someone can tell how live traffic gets scored, at what rate, by whom, and against what SLO.*

### 5. ISO/IEC 25059 and NIST AI RMF cross-check

- ISO/IEC 25059 completeness table (one row per characteristic, `in scope / deferred / not applicable`).
- NIST AI RMF Measure mapping: which categories each SLI serves.
- Note that Govern and Manage are the Governance peer's scope, not this plan's.

*Minimum bar: no ISO characteristic is blank; every RMF Measure category is either served by an SLI or explicitly out of scope.*

### 6. Deferred / handed off

Name the peer role and the interface for every criterion this plan does not measure itself:

- Judge-vs-human calibration methodology → `model-evaluation-engineer`.
- Benchmark-construction methodology → `model-evaluation-engineer`.
- Harm-model authoring / red-team dataset generation → `ai-risk-engineer`.
- Audit trails, regulator packaging, third-party evaluator interface → `ai-evaluation-engineer` (Governance family, level 25).
- Deep AI/ML security (eval-set exfiltration, judge supply-chain) → `ai-infra-security`.

*Minimum bar: no criterion is deferred by silence. Every deferral names a peer and an interface.*

### 7. Ship gates and rollback

- Pre-registered SLO thresholds for merge, canary, and full rollout.
- Rollback criteria for the online loop.
- Which SLIs are ship-blocking and which are watch-only.

*Minimum bar: a release engineer can wire this into the deploy pipeline without asking follow-up questions.*

### 8. Sign-off

A short table:

| Role | Name | Sign-off scope | Date |
|---|---|---|---|
| PM | ... | criteria coverage matches product intent | |
| ML engineer | ... | offline / online implementation is feasible on the stated model + scaffold | |
| Governance owner | ... | RMF Measure coverage is sufficient; peer delegations are correct | |
| AI eval engineer | ... | authorship, upkeep, revision cadence | |

*Minimum bar: named humans, named scopes, dated sign-offs.*

## Rules for the plan itself

- **Two pages.** If you are over, cut context, not decisions.
- **No unexplained acronyms.** Introduce SLI/SLO/RAG-triad/RMF/25059 on first use.
- **Every table has a footnote pointing at the module in this track that owns depth.** Reviewers get their questions answered without you being in the room.
- **Version and date the plan.** SLOs move; the plan should say when it moved and why.
- **Change control.** Which SLIs need re-sign-off if they change (usually every ship-gate SLI). Which do not (informational drift watches can move with just author sign-off).

## Anti-patterns

- **The "we will measure quality" plan.** No families, no SLIs, no gates. Impossible to sign because nothing is falsifiable.
- **The "kitchen sink" plan.** 40 criteria, none prioritised, all deferred to "later." A plan that ships everything ships nothing.
- **The "silent deferral" plan.** Half the criteria are quietly not addressed. Governance owners cannot sign this.
- **The "benchmark alignment" plan.** Only cites public benchmarks; nothing on the actual surface distribution. See chapter 05.

## Worked example: meeting summariser two-page plan (excerpt)

The full plan lives in the exercise repo; here is a compressed version of sections 1–2 and 6–7 to show the shape.

**1. Surface & scope.** The *Meeting Summariser* surface converts a Zoom transcript into a 200-word summary and an attributed action-item list, presented in the meeting recap email 5 minutes after the meeting ends. Runtime: single-shot generation with retrieval over the transcript and calendar metadata; guardrails inline. Owners: PM `<name>`, ML `<name>`, AI eval `<name>` (you), Governance `<name>`. Config: `summariser-prompt v3.2`, retriever `transcript-index v1`, model `frontier-tier-A pinned build 2026-05`, decoding `T=0.2`, guardrails `hr-policy-check v0.9`. Out of scope: the "chat about this meeting" follow-up feature (separate plan).

**2. Criteria.**

| # | Family | Criterion | SLI | SLO | Gate | Placement |
|---|---|---|---|---|---|---|
| 1 | Task success | Summary is ≈ 200 words | Words / summary in [170, 230] | 100% on fixture; ≥ 99% online | merge | both |
| 2 | Task success | Action-item list is present and non-empty when the meeting has any commitments | Judge score ≥ 4 on presence rubric | ≥ 98% offline; ≥ 95% weekly online | merge & canary | both |
| 3 | Groundedness | Every action item names an attendee who appears in the transcript | Rule check | 100% offline; ≥ 99% weekly online | merge | both |
| 4 | Groundedness | Attributed attendee verbally accepted the action item | Judge score ≥ 4 on attribution rubric, judge calibrated against 100-case gold set | ≥ 98% offline; ≥ 95% weekly online | canary & watch | both |
| 5 | Safety | No content from HR-tagged transcript segment surfaces | Rule + judge; HR-seeded fixture | 0 leaks offline; ≤ 0.5% FP on benign | merge | both |
| 6 | Safety | Prompt-injection via calendar-invite text does not cause tool call or leak | Injection-fixture pass rate (`mod-108`) | ≥ 99% offline; watch online | merge | both |
| 7 | Cost | Cost per generated summary within budget | $/summary at deploy config | ≤ $0.02 offline; ≤ $0.025 p95 online | merge | both |
| 8 | Latency | User sees the card within the promised window | End-to-end p95 < 60s | fixture pass; ≥ 99% weekly online | canary & watch | both |

**6. Deferred / handed off.**

- Full judge-vs-human calibration methodology for the attribution and safety rubrics — deferred to `model-evaluation-engineer`; interface is the calibration methodology packet in that track.
- Harm model for "sensitive HR" content — deferred to `ai-risk-engineer`; interface is the harm-model artefact tag `hr-sensitive-content v1`.
- Full injection-red-team dataset generation methodology — deferred to `ai-risk-engineer`; interface is `injection-suite-source v2`.
- Audit trail packaging, regulator-facing evidence, third-party evaluator interface — deferred to `ai-evaluation-engineer` (Governance family, level 25); interface is the release-assurance ledger.

**7. Ship gates and rollback.**

- Merge gates: rows 1, 2, 3, 5, 6, 7 above.
- Canary gates: rows 2, 4, 8 (48-hour canary at 5% traffic; pass criterion = SLO met at canary sample size).
- Rollback: any 24-hour weekly moving SLO breach on rows 2, 4, or 5; sev-2 page for row 5 breach or any injection-fixture regression.
- Change control: rows 5 and 6 SLIs cannot be relaxed without governance re-sign-off.

The plan continues with sections 3, 4, 5, and 8 in the same compressed shape. The full file is what the exercise builds.

## Summary

- The two-page plan is the artefact the whole module builds toward.
- **Eight sections**, in order: surface & scope, criteria, offline suite, online loop, ISO/RMF cross-check, deferrals, ship gates, sign-off.
- Every section has a **minimum bar** — meet it or the plan is unsignable.
- **Two pages**, dated, versioned, with change control on ship-gate SLIs.
- Avoid the four anti-patterns: "we will measure quality," "kitchen sink," "silent deferral," "benchmark alignment."

The four exercises in this module take you from a raw spec to a signable plan. The lab wires the plan into a running fixture harness and CI check that the rest of the track builds on.
