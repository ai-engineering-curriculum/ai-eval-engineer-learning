# exercise-04: Public Benchmark Shape-Template Audit

**Estimated effort:** 3 hours

## Objective

Pick a product-agent surface and audit three public agent benchmarks against it from chapter 06's list — SWE-bench Verified, τ-bench, WebArena, GAIA, AgentBench, Berkeley Function-Calling Leaderboard, or ToolBench. For each benchmark, decide whether to **accept as a shape template**, **adapt** (borrow some shape, replace most), or **reject**. Produce a shape-template adoption plan for every `accept` or `adapt` and a short stakeholder memo defending the three verdicts.

The exercise is deliberately smaller than mod-101's `exercise-04` (which audits leaderboards against the eval plan). Here the focus is narrower: the chapter-02 scorer, the chapter-03 rubric, the chapter-04 budgets, and the chapter-05 Inspect wrapping. The question is "does this benchmark's *shape* plug into *this module's scorer stack*?".

## Prerequisites

- Chapter 06 of this module.
- One concrete product-agent surface. Use the module's running support-agent surface if you do not have an authorised workplace one. Have your criteria table from mod-101 exercise-01 to hand.
- Time to read at least the abstract, methods section, and scoring description of each candidate benchmark's primary source. Links are in `resources.md`.

## Choose three benchmarks

Pick three from the list below. Choose on purpose — pick one obvious accept, one plausible adapt, and one obvious reject (the stakeholder exercise is the audit, not the easy case):

- **SWE-bench Verified** — <https://openai.com/index/introducing-swe-bench-verified/>, Jimenez et al. <https://arxiv.org/abs/2310.06770>. Code patches, programmatic verifier via project tests.
- **τ-bench** — Yao et al. <https://arxiv.org/abs/2406.12045>. Airline + retail; user simulator; DB-state verifier; `pass^k`.
- **WebArena** — Zhou et al. <https://arxiv.org/abs/2307.13854>. Five self-hosted sites; programmatic end-state scorer.
- **GAIA** — Mialon et al. <https://arxiv.org/abs/2311.12983>. 466 multi-tool questions, three difficulty levels, exact-match on final short answer.
- **AgentBench** — Liu et al. <https://arxiv.org/abs/2308.03688>. Eight environments; one scorer family per environment.
- **BFCL** — <https://gorilla.cs.berkeley.edu/leaderboard.html>. AST-based function-call evaluation; multiple categories.
- **ToolBench / ToolLLM** — Qin et al. <https://arxiv.org/abs/2307.16789>. 16k RapidAPI tools; LLM-as-judge for pass rate / win rate.

If your workplace is actively evaluating a benchmark not on this list, use it instead and cite it from the primary source.

## Requirements

Produce a single markdown file `benchmark-audit-<surface-slug>.md` with four sections.

### 1. Surface recap (five lines)

Restate the surface, its production task families, the tool catalog, the final-answer shape (natural language reply? structured record? state mutation?), and the chapter-03 rubric's weight profile. This is what each audit is anchored against.

### 2. Per-benchmark audit (three sections, one per benchmark)

For each chosen benchmark, fill the following seven-row table:

| # | Question | Answer |
|---|---|---|
| 1 | **Task schema.** Does the benchmark's `(input, reference)` shape match this surface's? | ... |
| 2 | **Scoring function.** Is the benchmark's scoring algorithm reusable here? Which chapter-02 or chapter-03 scorer slot does it fit? | ... |
| 3 | **Metric family.** Is the benchmark's reporting metric (pass@1, `pass^k`, pass-rate, win-rate) a sensible shape for this surface's rollout gate (chapter 04)? | ... |
| 4 | **Harness compatibility.** Does the benchmark ship an Inspect Task (or an easily-adaptable harness)? If not, what would the port cost? | ... |
| 5 | **Contamination.** Is the benchmark public enough that frontier models have seen it? What does that imply for interpreting the number? | ... |
| 6 | **Judge dependence.** Does the scoring depend on an LLM judge? If so, is the judge choice something you want to inherit? | ... |
| 7 | **What to leave behind.** The dataset, the leaderboard number, the implied task distribution — name the specific thing you are *not* adopting. | ... |
| — | **Verdict** | `accept` / `adapt` / `reject` |

Full-sentence answers, not yes/no. An honest "unknown, would need to run the benchmark once to tell" is a legitimate answer to any of the seven questions.

### 3. Shape-template adoption plans (one per `accept` or `adapt`)

For every `accept` and `adapt` verdict, write a compact adoption plan (10–15 bullets, half a page):

- **What is borrowed** — schema? scorer? metric? harness?
- **What is replaced** — the dataset, the tools, the reference trajectories, the judge configuration.
- **Where it plugs into the module's stack** — chapter-02 Level-N argument matcher? chapter-03 final-answer scorer slot? chapter-05 Solver? chapter-04 reporting?
- **Which `mod-10x` module owns the implementation depth** — `mod-103` (this one), `mod-105` (RAG), `mod-108` (safety), `mod-110` (fixtures), `mod-111` (cost / latency).
- **Fixture versioning plan** — which version lives in mod-110? how does it tie to the rubric version (mod-106's hash discipline)?
- **A smoke test** — the five historical trajectories you would score with the shape before rolling into CI.

### 4. Stakeholder memo (≤ 300 words)

A memo you would send to the stakeholder who proposed all three benchmarks (the PM, the model-evaluation-engineer peer, or the engineering director). Structure:

- One-sentence recap of the proposal.
- Three bullets, one per benchmark, each with the verdict and the one-sentence defence.
- For every `reject`, a **specific alternative** — usually one of the other two, usually adopted as a shape template.
- One-line next step (e.g., "I will have the shape-template plan for τ-bench on the retail task family ready by Thursday — happy to walk the plan through at the next review").

Tone: a sentence a PM reads to the end. Not an academic exchange.

## Starter guidance

- Do the reading first. Two paragraphs about a benchmark you have not read is not an audit; it is an opinion.
- Avoid "reject because it is not our distribution" as a universal answer. Pick your three so at least one is genuinely arguable. The purpose of the exercise is to build the discernment, not to collect three rejects.
- The rubric is: `accept` means you plug it in largely as-is (dataset-replacement aside); `adapt` means you borrow a specific component (the scorer, the harness, the metric) and discard the rest; `reject` means its shape does not match your surface at all.
- Be honest about contamination. GPT-era frontier models have seen SWE-bench; that is not a reason to reject the shape — the shape is reusable regardless — but it is a reason not to trust the leaderboard number as a baseline for your model's capability on *your* repo.
- Where the benchmark ships an Inspect-compatible Task (several of the chapter-06 benchmarks have community implementations), note the specific repo / package.

## Acceptance criteria

You are done when:

- Three benchmarks are audited across all seven questions with full-sentence answers.
- Each has a clear `accept` / `adapt` / `reject` verdict with a one-sentence defence.
- Every `accept` and `adapt` has a shape-template adoption plan that names the chapter-N slot it plugs into.
- Every `reject` has a specific alternative.
- The stakeholder memo is under 300 words and reads as something you could actually send to a PM.
- The file names the chapter-03 verdict taxonomy entries that each adopted shape would feed into (`fabricated_claim`, `extra_tool`, etc.).
- The audit is explicit about what *leaderboard number* you are not adopting for each benchmark — not just by implication.

## Stretch goals

- **Smoke-run.** Pick the `adapt` benchmark and actually run its shape on five historical trajectories from your surface (reuse exercise-01's scorer; adapt the final-answer scorer to the benchmark's shape). Report the pass-rate.
- **Judge replacement.** For a benchmark whose scoring is LLM-as-judge (ToolBench; some τ-bench variants), swap in a stub-judge from exercise-02 and document the delta on five trajectories.
- **Reporting-shape memo.** Write a 200-word note on how your rollout dashboard would change if you reported `pass^k` from τ-bench next to the chapter-03 `pass_rate`. Which is the one a PM should see first?
- **Public-benchmark delta.** For your `accept` or `adapt` benchmark, compute (or estimate) how your surface's shape-template score compares to the benchmark's current leaderboard best. Note what is *not* comparable (distribution, prompt-template choice, judge).

## What this exercise does *not* cover

You are not running the benchmarks on the frontier (`model-evaluation-engineer` peer owns the depth), you are not building the Inspect Task for the adopted shape (that is exercise-03's style, applied in `mod-106`'s CI loop), and you are not deciding model-choice on the resulting number (`mod-107`, `mod-111`). You are practising the auditing skill that gates whether a benchmark should be run at all.
