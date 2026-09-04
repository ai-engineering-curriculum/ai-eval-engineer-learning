# The Multi-Objective Eval Report: Four Columns, a Pareto View, a Pre-Registered Rule

## Motivation

The trade-off report is the artefact the whole module is building toward. Every other chapter — routing, replacement regression, app-altitude latency, per-feature cost — plugs a column or a filter into this shape. If the report shape is wrong, the downstream chapters produce numbers that never resolve into a decision.

The single most common failure of first-cut trade-off reports is that they collapse into a **weighted single-number score**: `0.5 * quality + 0.2 * -log(cost) + 0.2 * -log(latency) + 0.1 * safety`, argmax, done. The weights come from a Slack thread, the winner is a candidate that would have been rejected on any single axis, and no one can defend the outcome to product a week later because "why 0.5 not 0.6" has no answer.

The second most common failure is a **giant dashboard**: forty tiles, ten cohorts, five candidates, updated every 5 minutes, no signed decision. The dashboard exists for months; the shipping decision never happens because there is nothing to *sign*.

This chapter builds a report shape that avoids both failures. Four columns kept separate. A Pareto view that shows the frontier without collapsing it. A pre-registered decision rule that turns "we ran the numbers" into "we made the call the rule dictated." A signed artefact — not a dashboard.

## Core concepts

### The report at a glance

A multi-objective eval report is one page. Not one dashboard, not one BI project — one page. The rest is drill-down evidence.

The page has, at minimum:

- **Header block** — decision id, timestamp, eval-set name + hash, incumbent candidate id, list of candidate ids, cohort roster, pre-registered decision rule, signer name and role.
- **Main table** — one row per candidate; columns for quality (with per-cohort breakout), cost, latency, safety; the incumbent's row shaded; the Pareto-frontier candidates marked.
- **Pareto view** — a small-multiples plot: quality vs. cost, quality vs. latency, cost vs. latency; each candidate a point, the frontier highlighted.
- **Cohort quality view** — one row per cohort, one column per candidate; the cohort-preservation contract (from chapter 03) evaluated per cell.
- **Decision block** — the pre-registered rule quoted verbatim; the rule evaluated against the numbers; the decision (ship / hold / roll back); the signer's line to countersign.
- **Rollback trigger** — the online-loop metric (from mod-107) and threshold under which the decision reverses automatically; the runbook link (from mod-106 chapter 03).

Everything else — every 5-minute-updated tile, every cohort deep-dive, every histogram of judge scores — is *drill-down*. The one page is what product signs.

### The four columns and their sub-columns

Chapter 01 named them; the report shape pins them down.

| Column | Sub-columns |
|---|---|
| **Quality** | `mean` (never displayed alone); `p50 / p95 / p99` of the rubric score; per-cohort `mean` + `worst-cohort` value; pass-rate against a chapter-04 acceptance threshold; win-rate against the incumbent on the paired eval set (chapter 04 shape) |
| **Cost** | Cost per request `mean / p95`; cost per feature per active user per day; input / output / cache-read / cache-write / batch token counts and dollar totals; per-cohort cost mean (some cohorts have longer prompts / more tool calls) |
| **Latency** | TTFT `p50 / p95 / p99`; TPOT `p50 / p95 / p99`; end-to-end streaming `p50 / p95`; retry rate; tool-call round-trip median; per-cohort worst; hard-timeout rate |
| **Safety** | OWASP LLM Top-10 posture (chapter 05 of mod-108) — pass / fail per category, worst category; jailbreak-resistance score on the mod-108 red-team set; PII-leak rate on the mod-108 privacy suite; policy-violation rate from the online guardrails |

The report never displays a single-number roll-up. `mean` alone is banned; every `mean` is displayed with `p95` (and, for cost and latency, `p99`). The reason: means hide tails, and the trade-off report exists to make tails legible.

### Pareto frontier and dominance

The Pareto view is the intuition pump the numeric columns support.

- **Candidate A dominates candidate B** iff A is at least as good as B on every axis and strictly better on at least one. A dominating B means B is never the right choice — pick A.
- **The Pareto frontier** is the set of candidates that no other candidate dominates. Every frontier candidate is a legitimate shipping option. Off-frontier candidates are dominated — reject them.
- **The decision reduces from "which of N candidates" to "which of K frontier candidates."** K is usually 2 – 4. The pre-registered rule then picks one of the K.

The four-axis frontier is not always plottable in 2D. The Pareto small-multiples plot the report uses is three 2D projections — quality × cost, quality × latency, cost × latency — with safety shown as a marker colour or shape (a red X for "fails one OWASP category" beats any 2D placement). A candidate on the frontier of all three projections is unambiguously frontier; a candidate on the frontier of one projection but not the others may or may not be — the numeric column decides.

Two special cases the shape has to handle:

- **A single-axis winner that is safety-red.** Quality-great, cost-great, latency-great, safety-fail-on-one-OWASP-category. Do not silently drop; the report marks it "excluded — safety gate" and shows *why*. Product needs the signal that the candidate exists but is off the table.
- **A candidate that dominates on three axes but has a policy blocker (e.g., cannot be self-hosted where regulation requires it).** Same treatment — mark "excluded — deployment gate" with the reason. The trade-off report is honest about what was ruled out and why.

### Cohort breakouts as first-class columns

Chapter 01 named the "average quality" anti-pattern. The report shape stops it by requiring cohort breakouts as first-class report elements.

A cohort is a pre-declared, request-time-populated tag on every trace. Typical cohorts (declare per product surface — no universal list works):

- **Locale** — `en-US`, `en-GB`, `es`, `de`, `ja`, `zh`, ...
- **Tenant tier** — `free`, `pro`, `enterprise`, `regulated`.
- **Query type** — `factual`, `code`, `creative`, `multi-turn`, `tool-heavy`.
- **Input length band** — `short (< 512)`, `medium (512 – 4k)`, `long (4k – 32k)`, `long-context (> 32k)`.
- **User segment** — `first-time`, `weekly-active`, `power-user`.

The rule: **any cohort a stakeholder would ask about after a regression is a cohort the report shows before the regression happens.** If the enterprise tenant tier's regression would prompt a Slack from the CS lead, the enterprise tier is a cohort. If nobody would ever ask about the `zh` cohort's quality, drop it from the report — but do not drop it from the store; a future stakeholder might.

Cohorts are the axis of the cohort-preservation contract chapter 03 defines. The report shape here is: one row per cohort × candidate, one cell showing quality mean and worst-cohort-cell deltas versus the incumbent, and a red / yellow / green flag against the contract's δ.

### The pre-registered decision rule

The report's most important element is the smallest — a single paragraph written *before* the numbers are seen.

Format:

```
Pre-registered decision rule (written 2026-08-14, signed by <PM>, <Eng>)

Ship candidate <C> if all of:
  1. Quality on cohort <critical_cohort> drops by no more than 0.5 rubric points
     versus incumbent on the frozen <eval_set@hash>.
  2. No cohort's quality drops by more than 1.0 rubric points versus incumbent.
  3. Cost per request (mean) drops by at least 30% versus incumbent.
  4. Streaming p95 does not increase by more than 200 ms versus incumbent.
  5. Zero OWASP LLM Top-10 category regressions versus incumbent.

Hold if any 1–5 is unmet but the gap is < 25% of the threshold. Reject otherwise.

Rollback trigger (post-ship): if the mod-107 online-loop faithfulness rubric on
cohort <critical_cohort> drops by more than 0.5 points over a 24-hour window
against the pre-ship baseline, auto-rollback via flag <flag_name>.
```

Three properties matter:

- **Written before the numbers were seen.** If the rule is written after the run, the thresholds silently move to bracket the winning candidate. Pre-registration is the anti-motivated-reasoning contract.
- **Signed by the product owner *and* the engineering owner.** Two signatures because one signature is a foot-gun. The engineering owner catches "cost per request" mis-spec (mean vs. p95); the product owner catches thresholds that would kill features the org cares about.
- **Rollback trigger is part of the rule.** The decision rule is not "ship." The decision rule is "ship, with this trigger." Chapter 04 of mod-107 defined the online-loop signals; this rule points at them.

Un-pre-registered decision rules are how a report becomes a rubber stamp. The exercise for this chapter forces the rule to be committed to the repo before the report is generated.

### Tie-breakers

Even a well-shaped report will sometimes present two frontier candidates that the pre-registered rule accepts equally. Tie-breakers, in the order the module recommends:

1. **Cohort-preservation margin.** Prefer the candidate with the smaller worst-cohort regression. A frontier candidate whose worst cohort is 0.3 points below incumbent beats one whose worst cohort is 0.9 below — even if both are within the 1.0 threshold.
2. **Operational simplicity.** Prefer the candidate the on-call rotation has more experience with. A frontier candidate on the incumbent vendor beats a frontier candidate that requires onboarding a new vendor unless the second dominates on multiple axes.
3. **Safety headroom.** Prefer the candidate with the larger jailbreak-resistance margin. All else equal, more safety headroom is worth having.
4. **Cost-trajectory.** Prefer the candidate on a decreasing-cost trajectory — an OSS model whose serving cost the fleet team is actively bringing down beats a vendor model on the "prices are firm" side of the market, absent other signal.
5. **Latency headroom for tool-call growth.** Prefer the candidate with more latency headroom against the product's roadmap — if the next feature adds two tool calls, the 400-ms-headroom candidate ships even if the 200-ms-headroom candidate is otherwise equal.

Tie-breakers are pre-registered too — they should sit alongside the decision rule. A tie-breaker invented at report-signing time is another motivated-reasoning surface.

### Who signs

The trade-off report is a joint artefact. The eval team produces it; product signs; engineering countersigns. In practice:

- **Producer.** The AI Evaluation Engineer (this role). Owns the report shape, the numbers, the pre-registration.
- **Signer — product.** Product manager for the surface. Owns the "we accept these trade-offs against the roadmap" call. Their signature makes the report a shipping authorisation.
- **Signer — engineering.** Engineering lead for the surface. Owns the "we can operate this in production" call — including the rollback trigger and the on-call implications.
- **Consulted — safety / governance / legal.** For any candidate with a safety-column change, or a jurisdictional change (new vendor region, new deployment model), the consultation is on the record. Consulted ≠ signed; the veto power depends on the org's governance shape (mod-108 chapter 05 covers this in more depth).

The report has a signature block. Missing signatures block the ship. This is the same discipline mod-106 chapter 03 applied to the release runbook, extended to the multi-axis decision.

### The rollback trigger

The pre-registered rule includes a rollback trigger — the condition under which the shipping decision reverses. This is the single most-often-skipped element of trade-off reports and the single most valuable.

Shape:

- **A monitored metric.** From the mod-107 online loop — usually a rubric score on a cohort, or a drift metric, or a safety-guardrail rate.
- **A threshold and window.** "Drops by more than 0.5 rubric points over a 24-hour window" is a threshold and window. "Looks bad" is not.
- **An automatic action.** A flag flip, a router-tier restriction, an A/B ramp-down. Not "we page someone." The rollback is *automatic*; the page is the *notification*.
- **A grace period on false-positives.** The first rollback in a 7-day window auto-fires; the second requires a human confirm to avoid oscillation. Mod-107's SLO / error-budget shape applies.

A shipping decision with no rollback trigger is a shipping decision that trusts the pre-ship numbers to hold indefinitely. Chapter 04 of mod-107 covered why that trust is wrong; this report's rollback trigger is how the trust is bounded.

### Anti-patterns to avoid

**"Single-number score."** Chapter 01 named it; the report shape forbids it. If the org insists on a single-number roll-up for an exec view, provide one — labelled "informational, not the decision" — with a footnote naming the un-defendable weights.

**"Vanity-cohort report."** Every cohort in the report is one someone would ask about. A report with 15 cohorts nobody asks about signals cohort inflation — the important cohort will be diluted. Cut cohorts to the ones tied to a stakeholder.

**"The dashboard IS the report."** A dashboard is a debugging surface; a report is a signed artefact. If the org's process is "check the dashboard before every deploy," there is no report — no timestamp, no lineage, no signature, no decision. Ship the report as a static Markdown / PDF / HTML file, hashed and stored; the dashboard is the drill-down.

**"Report by consensus."** The report shape is not negotiated per decision. If every trade-off ships with a different column set, the reports do not compare and the org loses institutional memory. Fix the shape once (this chapter); apply it uniformly.

**"Cost as a footer."** Cost buried at the bottom, or in a separate document. This is how the "we shipped the frontier model because the eval-team dashboard was quality-only" story starts. Cost is a top-line column, side-by-side with quality.

**"Silent safety column."** Safety shown as a green tick if it happens to be green; hidden if it is not. If safety fails, the report shows the failure and the candidate is marked excluded — not omitted. Chapter 05 of mod-108 covers the discipline.

### A minimal report walkthrough

To make the shape concrete, a stripped-down example. The eval team is picking between three candidates — the incumbent (`inc-support-frontier-v3`), a mid-tier model (`cand-support-mid-v1`), and a routed policy (`cand-support-routed-v1`) — for the support-bot.

```
# Trade-off report — support-bot model selection

decision_id:      TR-2026-08-19-support-bot
eval_set:         support-eval-v14  (hash sha256:f2c...c13)
timestamp:        2026-08-19T14:03:11Z
cohorts:          en-US, en-GB, es, de, ja, enterprise, regulated,
                  short-input, long-input
incumbent:        inc-support-frontier-v3
candidates:       cand-support-mid-v1, cand-support-routed-v1
pre_registered:   rules/support-bot-2026-08-14.md  (sig: PM=alex, ENG=jamie)
signer:           <pending>

## Main table

| candidate                   | quality mean / worst-cohort | cost / req  | ttft p95 | streaming p95 | safety |
|-----------------------------|-----------------------------|-------------|----------|---------------|--------|
| inc-support-frontier-v3     | 4.22 / 3.71 (regulated)     | $0.0184     | 620 ms   | 3.9 s         | pass   |
| cand-support-mid-v1         | 3.98 / 3.15 (regulated)     | $0.0059     | 380 ms   | 2.1 s         | pass   |
| cand-support-routed-v1      | 4.19 / 3.68 (regulated)     | $0.0091     | 470 ms   | 2.8 s         | pass   |

Pareto frontier: {inc-support-frontier-v3, cand-support-routed-v1}
(cand-support-mid-v1 dominated by cand-support-routed-v1 on quality & safety-margin)

## Cohort quality vs. incumbent (Δ rubric points; contract δ = -1.0)

|                | en-US | en-GB | es   | de   | ja   | enterprise | regulated | short | long  |
|----------------|-------|-------|------|------|------|-----------|-----------|-------|-------|
| mid-v1         | -0.14 | -0.19 | -0.31| -0.28|-0.42 | -0.51     | -0.56 ⚠   | -0.11 | -0.67 |
| routed-v1      | -0.02 | -0.03 | -0.04| -0.05|-0.08 | -0.06     | -0.03     | -0.01 | -0.09 |

## Decision

Pre-registered rule (verbatim):
  Ship candidate C if:
    (1) quality on regulated cohort drops ≤ 0.5 rubric points   [mid: -0.56 ✗; routed: -0.03 ✓]
    (2) no cohort drops > 1.0 points                            [mid: ✓ (max -0.67); routed: ✓]
    (3) cost per request drops ≥ 30%                            [mid: 68% drop ✓; routed: 51% drop ✓]
    (4) streaming p95 does not increase > 200 ms                [mid: -1.8 s ✓; routed: -1.1 s ✓]
    (5) zero OWASP LLM Top-10 category regressions              [both ✓]

Result:
  cand-support-mid-v1     — REJECT   (fails rule 1: regulated cohort -0.56 > 0.5)
  cand-support-routed-v1  — SHIP     (all rules pass; frontier)

Rollback trigger:
  If mod-107 online-loop faithfulness rubric on regulated cohort drops > 0.5 points
  over a rolling 24 h window vs. pre-ship baseline, auto-rollback via flag
  support_router_v1 → 0%. Runbook: runbooks/support-router-rollback.md
```

The report is one page. The Pareto plot and the cohort matrix drill-downs sit behind links. The rule is quoted verbatim. The decision names *both* the ship and the reject candidate — the reject line matters because it prevents "why not the cheap one" from re-litigating outside the report.

### Where each subsequent chapter plugs in

- **Chapter 03 (routing).** Adds the cohort-preservation contract as a first-class rule in the pre-registered set; contributes the router-error separation to the quality column.
- **Chapter 04 (replacement regression).** Provides the paired-comparison methodology behind the cohort-quality Δ matrix; formalises the ship / hold / reject decision matrix.
- **Chapter 05 (app-altitude latency).** Provides the TTFT / TPOT / streaming-p95 numbers in the latency column; adds the reconciliation with the model-altitude MLPerf-shaped benchmark as a footnote.
- **Chapter 06 (per-feature cost accounting).** Provides the cost-column numbers with input / output / cache-read / cache-write / batch splits; adds the per-feature attribution.
- **Chapter 07 (delegation).** Defines what the report reads from the peers (frontier-model quality profile, MLPerf serving curves) and what it publishes back (app-altitude cohort regressions).

Each downstream chapter's exercise adds one column's worth of numbers to this shape. Exercise 01 ships the shape itself.

## Summary

- The report is **one page** — header, main table (four columns, per-cohort breakouts, Pareto marks), Pareto small-multiples, cohort matrix, pre-registered decision rule, decision block, rollback trigger. Everything else is drill-down.
- The **four columns** are quality, cost, latency, safety — each with sub-columns (`mean` never alone; always with `p95`). No single-number roll-up.
- The **Pareto frontier** is the subset of candidates no other candidate dominates. The decision reduces from N candidates to the K on the frontier, then the pre-registered rule picks from K.
- **Cohort breakouts are first-class columns**, not drill-downs. Any cohort a stakeholder would ask about post-regression is a cohort the report shows pre-regression.
- The **pre-registered decision rule** is written before the numbers are seen, signed by product *and* engineering, and includes a rollback trigger tied to a mod-107 online-loop metric.
- **Tie-breakers** are pre-registered too: cohort-preservation margin > operational simplicity > safety headroom > cost trajectory > latency headroom for roadmap growth.
- The report is a **decision artefact**, not a dashboard: timestamped, lineage-pinned, signed, hashed. The dashboard is the drill-down.
- **Anti-patterns**: single-number score, vanity-cohort report, dashboard-as-report, report-by-consensus (shape re-negotiated per decision), cost as a footer, silent safety column.

Chapter 03 opens the first specialised report the shape carries — model routing with cohort preservation.
