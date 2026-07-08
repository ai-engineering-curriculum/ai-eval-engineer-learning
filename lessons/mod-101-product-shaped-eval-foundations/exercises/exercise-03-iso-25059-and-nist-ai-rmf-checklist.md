# exercise-03: ISO/IEC 25059 and NIST AI RMF Checklists

**Estimated effort:** 2 hours

## Objective

Apply two external frameworks to the criteria and placement work you did in exercises 01 and 02 as a completeness pass. Produce a filled-in ISO/IEC 25059 completeness table, a NIST AI RMF Measure mapping, and (for a generative surface) a Generative AI Profile (AI 600-1) risk sweep. The goal is to catch families of criteria the spec was silent on and to state peer-owner boundaries explicitly, not to author a governance program.

## Prerequisites

- Exercises 01 and 02 completed for at least one surface.
- Chapter 04 of this module.
- Access to the public reference materials — you do not need paid ISO access to do this exercise, but you must cite what you consulted.

## Requirements

Produce a single markdown file `standards-checklist-<surface-slug>.md` containing three sections.

### 1. ISO/IEC 25059 completeness table

Walk the AI-system quality characteristics from ISO/IEC 25059:2023 (an extension of ISO/IEC 25010's product-quality model). Public introductions of ISO/IEC 25010 are widely available; the AI-specific extensions in 25059 add characteristics such as functional adaptability, user controllability, and transparency, applied to AI behaviour.

<!-- needs-research: verify the currently published characteristic list of ISO/IEC 25059:2023 against the ISO catalogue in the next research cycle; refresh this exercise's checklist if the standard has been amended. -->

Produce a table with one row per characteristic, columns:

- `characteristic`
- `in-scope for this plan` — `yes` / `no` / `deferred`
- `criterion(s) from exercise-01 that cover it` — row numbers from your criteria table
- `SLI` — copy or reference from your exercise-01 table
- `if deferred, to whom` — peer role and interface
- `if not applicable, one-sentence rationale`

**Rules:**

- Every row is answered. No blanks.
- If a characteristic has *no* criterion in exercise-01 and is `in scope`, you have found a gap — add a row to your criteria table and mark it here.
- If a characteristic is deferred, name the peer role from chapter 04's mapping (this track / `ai-risk-engineer` / `ai-evaluation-engineer` (Governance) / `model-evaluation-engineer` / `ai-infra-security` / product analytics / etc.).

### 2. NIST AI RMF Measure mapping

Using the [NIST AI Risk Management Framework 1.0 (AI 100-1)](https://www.nist.gov/itl/ai-risk-management-framework), produce a mapping in two parts:

- **Function scope statement.** One paragraph declaring that this plan is a **Measure**-function artefact and naming the peer role or team that owns **Govern** and **Manage** in your (real or hypothetical) organisation.
- **Measure-category mapping.** For each RMF Measure category, a row with columns `category`, `which SLIs serve it`, `if none, why the plan is complete without one`. Use the RMF publication for the current category list; do not paraphrase from memory.

**Rules:**

- Every Measure category is either served by ≥ 1 SLI or has an explicit "not applicable — reason."
- Every reference to Govern or Manage names a real or hypothetical peer role, not "someone else."

### 3. Generative AI Profile (AI 600-1) risk sweep

For each risk in the [Generative AI Profile (AI 600-1)](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile) — confabulation, dangerous / harmful content, misuse, IP, information integrity, information security, obscene content, value-chain, etc.:

- Is this risk plausibly present on your surface? `yes / partial / no`
- If yes / partial, which criterion in your table covers it? If none, add one and mark it here.
- If no, one-sentence rationale.

Cross-reference each risk to the module in this track that owns the depth (`mod-105` for RAG-triad/confabulation, `mod-108` for injection / harmful-content on the surface, etc.).

### 4. Diff report

At the end of the file, list the **new criteria you added to your exercise-01 table because of this pass** — one bullet per addition, one sentence per bullet. If you added nothing, that is a defensible outcome; say why the spec was already complete.

## Starter guidance

- Do the frameworks in the order given: 25059 first (per-characteristic completeness), then RMF Measure (program-level scope), then AI 600-1 (GenAI-specific risks). This is the order chapter 04 recommends.
- If a characteristic looks like it duplicates one you already covered, do not force a new criterion; instead, in the "criterion(s) from exercise-01 that cover it" column, cite the row you already have.
- The two failure modes to avoid are **checklist theatre** (every row `yes / in scope` with no real depth) and **checklist paralysis** (every row deferred). The plan should live somewhere sensible in between.
- Distinguish clearly between "measured on my surface" (what you own) and "authored by a peer role" (what you defer). Both are legitimate answers.

## Acceptance criteria

You are done when:

- The 25059 table has one row per characteristic, no blanks, and every "no" or "deferred" is justified.
- The RMF section declares the Measure-only scope and maps every Measure category.
- The AI 600-1 sweep addresses every listed risk.
- The diff report is present, whether it lists new criteria or explains their absence.
- Every peer-role deferral is real (matches the ownership map in chapters 01 and 04).
- The whole file, read alongside exercises 01 and 02, would satisfy a governance owner asking "did you consider X?" for standard values of X.

## Stretch goals

- **Delta to a second framework.** If your organisation is subject to the EU AI Act (Regulation 2024/1689) or ISO/IEC 42001:2023, add a short annex mapping your Measure-category coverage to those frameworks' obligations — noting that authorship of an AI Act or 42001 evidence packet is the Governance peer's scope, not yours.
- **Model card slice.** Draft the two-paragraph "evaluation" slice of a product-side model card (Mitchell et al. 2018, *Model Cards for Model Reporting*) using only the criteria and SLIs you built. Note where you would ask the Governance peer to wrap it.

## What this exercise does *not* cover

You are not authoring a NIST-aligned or ISO-aligned assurance program (Governance peer's scope) and you are not authoring a harm model (`ai-risk-engineer`'s scope). You are using the two frameworks as **checklists** to make sure your plan does not have obvious gaps.
