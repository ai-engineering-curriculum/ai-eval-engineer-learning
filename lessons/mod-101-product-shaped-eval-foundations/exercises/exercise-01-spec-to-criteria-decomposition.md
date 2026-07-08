# exercise-01: Spec-to-Criteria Decomposition

**Estimated effort:** 2 hours

## Objective

Take a product spec written for a real AI application surface, tag it for the four claim types from chapter 02, and produce a criteria table with numeric SLIs across the five families (task success, groundedness, safety, cost, latency). Practise the "so what" test on every row before you commit.

## Prerequisites

- Chapters 01 and 02 of this module.
- A quiet hour with no distractions.

## Choose a spec

Pick one — do not do all three, but do read the other two so you have a comparison later:

1. **"Draft the follow-up email from this sales call."** Runtime: retrieval over the call transcript + one tool call (CRM lookup). Constraints: US business hours only, 3-paragraph body, cite one transcript quote, no forward-looking financial claims. Latency budget: p95 < 8s to first paragraph. Cost budget: $0.015 per generated email.
2. **"Summarise this pull request."** Runtime: retrieval over PR diff + linked issue text. Constraints: 8-bullet list of changes; distinguish behaviour changes from refactors; do not disclose author names; must not surface secrets from `.env`-like files. Latency budget: p95 < 15s. Cost budget: $0.005 per PR summary.
3. **"Answer this support question with docs and open a ticket if unresolved."** Runtime: RAG over public docs + one tool (`open_ticket`). Constraints: cite at least one doc URL; refuse account-modification requests; open ticket only when confidence < threshold. Latency budget: p95 < 20s end-to-end. Cost budget: $0.03 per resolved conversation.

If your workplace has a real spec you can work from *with your employer's authorisation*, use that instead. Redact anything sensitive before you commit files.

## Requirements

Produce a single markdown file `criteria-table-<surface-slug>.md` in your working repo containing:

1. **The tagged spec.** The full spec text with inline tags — `[intent]`, `[grounding]`, `[constraint:safety|cost|latency]`, `[distribution]` — on every substantive sentence. Untagged sentences are a real answer only if you say why they carry no eval signal.
2. **The claims → criteria ladder** for the three most load-bearing tagged sentences. Show each step: **claim → criterion → SLI → SLO → gate class → placement (offline/online/both)**.
3. **The criteria table.** At minimum one row per criteria family (task success, groundedness, safety, cost, latency). Each row has: `#`, `family`, `criterion`, `SLI`, `SLO (window + population)`, `gate class`, `placement`. Add a `notes` column for any deferral to a peer role.
4. **The "so what" test log.** For each row, one sentence answering *actionability* ("if this SLI drifts, what will the team do?") and one answering *attribution* ("if this SLI moves, can we tell which change caused it?"). Rows that fail either test are dropped from the table with an explanation.
5. **A one-paragraph gap note** identifying at least one family where the spec was silent and the corresponding question you would ask the PM.

## Starter guidance

- Read chapter 02 with your spec next to you. Do not try to hold both in memory.
- Do the tagging pass first, in one sitting. Do not evaluate quality yet — just tag.
- When you feel like writing "quality" or "accuracy" as a criterion, stop and re-read the "falsifiable and numeric" rule.
- Every SLO needs a **window** and a **population**. "≥ 92%" is not an SLO; "≥ 92% on weekly-sampled English traffic" is.
- If you end up with more than ~10 rows, you are probably not consolidating. If you end up with fewer than ~5, you have probably skipped a family.
- Do not pick SLO numbers by feel. Say "TBD, will pre-register with PM" if you do not have data — an honest TBD is better than a fake number.

## Acceptance criteria

You are done when:

- Every substantive sentence of the spec is tagged, with untagged sentences justified.
- At least three ladders (claim → criterion → SLI → SLO → gate → placement) are shown in full.
- Every criteria family has at least one row.
- Every SLI is numeric. Every SLO names window and population.
- Every row passes the "so what" test, with the test result visible in the log.
- At least one gap in the spec is flagged in the paragraph note.
- The whole file reads in under 10 minutes for someone who has read chapters 01 and 02.

## Stretch goals

- Redo the exercise for a second surface from the list and produce a two-column comparison of the resulting criteria tables. Notice how much is shared and how much is surface-specific.
- Add a **cohort dimension** to two of your SLOs: split the target by a plausible cohort (traffic source, locale, feature flag) and defend the split.
- Draft the PM-facing follow-up email for your gap note: single paragraph, one specific ask.

## What this exercise does *not* cover

You are not deciding offline vs online here (that is exercise-02), and you are not writing the two-page plan yet (exercise-04 and the lab). Focus on the ladder and the table.
