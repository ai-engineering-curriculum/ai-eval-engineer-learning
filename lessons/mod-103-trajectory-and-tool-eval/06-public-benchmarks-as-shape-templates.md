# Public Agent Benchmarks as Shape Templates (Not Product SLOs)

## Motivation

Chapter 01 drew the line: a public benchmark measures *the model's capability on a stylised task family under a fixed harness*; a product trajectory scorer measures *your surface's actual production trajectories against your product's SLOs*. The two numbers answer different questions, and conflating them is one of the most common failure modes in agent-eval work (mod-101 chapter 05 enumerates eight such failure modes).

That said, public benchmarks are not useless to a product eval. They are *shape templates*: the research community has already agreed on a task schema, a scoring function, and a reference trajectory format for whole task families. If your product's trajectories look like a tool-and-user-interaction loop (support, retail, financial customer service), τ-bench's schema is a tested starting point. If your trajectories look like a browsing agent working against web pages, WebArena's scoring is a tested starting point. The pattern this chapter develops: **borrow the shape, replace the content**. You adopt the schema, the metric family, and (sometimes) the harness; you discard the task distribution and the leaderboard number.

This chapter walks seven public benchmarks — SWE-bench Verified, τ-bench, WebArena, GAIA, AgentBench, BFCL, ToolBench — and for each specifies what to borrow, what to replace, and what to leave on the leaderboard. The exercise at the end of the module (`exercise-04`) is the audit the chapter builds up to.

## Core concepts

### What "shape template" actually means

Four things a public benchmark provides that your product's trajectory eval can reuse:

- **Task schema.** The shape of the input, the shape of the reference, the set of tools the agent is permitted. A schema is one or two typed structures; it has nothing to do with the particular tasks in the benchmark.
- **Scoring function.** The algorithm that turns an observed trajectory into a verdict. For SWE-bench, that is "apply the patch, run the test suite, read the exit code". For BFCL, that is an AST comparison between the predicted function call and the reference. The scoring function is often more valuable than the dataset.
- **Harness.** The runner, the agent loop, the retry policy, the logging schema. Chapter 05 adopted Inspect as the module's harness; several of the benchmarks below ship Inspect-compatible task packages.
- **Reporting shape.** Per-slice reporting, aggregation metric (pass@1, pass@k, pass^k, success rate), and the vocabulary used to describe failures. Reusing the reporting shape makes a conversation with a peer-team ML engineer legible: "our τ-shape pass^4 is 0.68 on retail-order-status, baseline is 0.72" is a sentence a model-evaluation-engineer peer understands immediately.

Three things a public benchmark provides that you should *not* reuse as-is:

- **The task distribution.** The benchmark's inputs are drawn from the researchers' chosen distribution, not yours. Replace them with samples from your production traffic (mod-110's fixture store), minus PII (mod-102 chapter 06).
- **The leaderboard number.** The headline score describes the model's capability on the benchmark's tasks. It is not your surface's pass-rate; do not pretend it is.
- **Contamination assumptions.** Most public benchmarks are public enough that frontier models have seen them. Your production traffic has not been in anyone's training corpus (unless you were careless with logs; see mod-102 chapter 06). Treat the shape as evergreen; treat any specific score as a weak prior at best.

### SWE-bench and SWE-bench Verified

**Benchmark.** Jimenez et al., "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?" (<https://arxiv.org/abs/2310.06770>, 2023). 2,294 issue + repository + fix-pull-request triples, mostly from twelve popular Python projects. The task: given an issue description and the repository state, produce a code patch that resolves the issue. SWE-bench Verified (OpenAI, 2024 — <https://openai.com/index/introducing-swe-bench-verified/>) is a 500-task human-reviewed subset addressing specification clarity and test coverage.

**Shape to borrow.** The programmatic verifier is the gold-standard pattern. A patch is applied; the project's own test suite is run inside a reproducible sandbox; the exit status is the verdict. There is no judge and no prompt-template dependence; the scoring function is deterministic. For any internal "fix this bug" or "implement this spec" agent surface, borrow the pattern directly: scoring is `apply + test + exit`.

**Shape to replace.** Replace the twelve open-source Python repos with your own codebase (or a redacted slice of it). Replace the issue text with your internal ticket text. Keep the "apply the patch and run the project's own tests" verifier structure. Keep the per-repo isolation (Docker sandbox) that makes the verifier deterministic.

**Shape not to adopt.** The leaderboard number. SWE-bench's leaderboard is a model-capability signal on open-source Python; it is not a signal on *your* bug-fix agent on *your* codebase.

**Where it plugs into this module.** The chapter-03 final-answer scorer, "programmatic verifier" row, is exactly this. The chapter-05 Inspect Task would wrap the apply/test loop as a Scorer; the budget scorer (chapter 04) treats the time to run the project's tests as part of trajectory latency.

### τ-bench (tau-bench)

**Benchmark.** Yao et al., "τ-bench: A Benchmark for Tool-Agent-User Interaction" (<https://arxiv.org/abs/2406.12045>, 2024). Two domains — airline and retail — with a human-shape user simulator, a toolset per domain, and ground-truth database state at the end of the dialogue. The scoring compares the final database state to the reference; `pass^k` reports the fraction of tasks the agent completes on **all** of k independent runs (a reliability metric, not a mean).

**Shape to borrow.** Three things:

- The **user-simulator** architecture — a separate model plays the user, so trajectories are multi-turn interactions rather than one-shot prompts. For any surface where the production trajectory is multi-turn (support, triage, intake), the simulator pattern is directly reusable; mod-109 chapter on human review notes when a real user is needed instead.
- The **database-state scorer** — compare the end state (CRM, orders DB) to the reference rather than the natural-language reply. This is the trajectory-eval analogue of a programmatic verifier for state-mutating tools.
- The **`pass^k`** metric. Reliability-over-k-runs is the right reporting shape for agents that are sensitive to sampling noise; a single pass@1 hides tail behaviour. Carry a `pass^k` roll-up next to pass-rate when the agent's temperature is non-zero.

**Shape to replace.** The airline and retail domains, their tools, and their database schemas. Use your own tools and your own database schema. Keep the simulator template.

**Where it plugs in.** The chapter-05 Solver maps to the τ-bench agent; the chapter-04 budget scorer tracks the per-trajectory cost of running both the agent and the user-simulator. The database-state Scorer is a direct Inspect Scorer.

### WebArena and friends

**Benchmark.** Zhou et al., "WebArena: A Realistic Web Environment for Building Autonomous Agents" (<https://arxiv.org/abs/2307.13854>, 2023). 812 tasks across self-hosted versions of a shopping site, a GitLab, a Reddit-like forum, a map tool, and a wiki. Scoring is programmatic — URL-match, element-presence, or value-match — against ground-truth outcomes. Related: VisualWebArena, WorkArena, Mind2Web.

**Shape to borrow.** Programmatic end-state scoring for browsing / UI-navigation agents. A browsing agent's trajectory is a sequence of `(click, type, scroll, screenshot)` observations; the end-state verdict is "did the agent reach the right URL / fill the right form / click the right button". This maps cleanly onto a chapter-02 reference trajectory in DAG form plus a chapter-03 programmatic terminal verifier.

**Shape to replace.** The five self-hosted websites. Use your product's own web surface or a staging environment. Keep the "programmatic verification over the final page state" scorer family.

**Where it plugs in.** For any UI-navigation agent surface, the chapter-05 Task's Solver drives a browser (via Playwright or similar), the chapter-02 scorer compares the click / type sequence against the reference, and the chapter-03 final-answer scorer runs the WebArena-style programmatic end-state check.

### GAIA

**Benchmark.** Mialon et al., "GAIA: A Benchmark for General AI Assistants" (<https://arxiv.org/abs/2311.12983>, 2023). 466 real-world questions requiring multi-step tool use (web browsing, document reading, code execution, image understanding), split across three difficulty levels. Scoring is exact-match on the final short-form answer; the trajectory is unscored.

**Shape to borrow.** The multi-level difficulty taxonomy — the same task family at Level 1 (one tool, short chain), Level 2 (two–three tools, dependencies), Level 3 (many tools, long chain, requires reasoning) — is a template for stratifying your product's task family: easy / medium / hard tiers you can report on separately. A trajectory scorer that averages across all three loses signal; stratified pass-rate preserves it.

**Shape to replace.** The 466 questions and their topic distribution. Author your own stratified samples per task family.

**Shape to be careful with.** GAIA scores only the final answer, with no trajectory scorer. If you borrow only this shape, you discard everything chapters 02 and 04 built. Use GAIA's taxonomy *in combination with* this module's trajectory scorer, not as a substitute.

### AgentBench

**Benchmark.** Liu et al., "AgentBench: Evaluating LLMs as Agents" (<https://arxiv.org/abs/2308.03688>, 2023). A multi-environment benchmark: operating-system commands, database queries, knowledge-graph queries, card game, lateral thinking puzzles, household-task simulation, web shopping, web browsing. Each environment has its own scorer and its own set of tools.

**Shape to borrow.** The per-environment taxonomy — each environment is a *task family* with its own tool set and its own scoring function. For a product with multiple distinct agent surfaces (support, triage, order-management), the AgentBench pattern of "one scorer family per environment" maps onto "one scorer family per task family".

**Shape to replace.** The eight environments themselves. Keep the "distinct task families → distinct scorer families, aggregated at the top" structure.

**Where it plugs in.** The chapter-05 convention of one `Task` per task family, each with its own `Sample` dataset and its own Scorer stack, is already AgentBench-shaped. The explicit mapping is: your `support_agent_order_status`, `support_agent_refund_request`, `support_agent_return_label` Tasks are each one AgentBench-style environment.

### Berkeley Function-Calling Leaderboard (BFCL)

**Benchmark.** <https://gorilla.cs.berkeley.edu/leaderboard.html>. Evaluates function-calling correctness with an **AST-based** evaluator: the predicted function call and the reference call are both parsed to AST and compared structurally rather than textually. Multiple categories (simple, multiple, parallel, parallel-multiple, relevance detection, and a live subset) stress different aspects of function-calling behaviour. <!-- needs-research: BFCL has v1 / v2 / v3; confirm the current version's category list on the next cycle and update. -->

**Shape to borrow.** The AST evaluator is the chapter-02 Level-4 argument matcher. For any tool whose arguments are themselves code (SQL, Python, shell), BFCL's AST comparison is the right pattern. The categorical taxonomy — "does the model know *when* to call a function" (relevance detection) vs "can it call the right one" vs "can it call several in parallel" — is a useful decomposition when debugging a tool-selection regression.

**Shape to replace.** BFCL's own function catalog. Use your product's own tool catalog in the same AST-shaped comparator.

**Where it plugs in.** The chapter-02 argument matcher's Level-4 slot maps to BFCL's AST evaluator directly. The chapter-04 budget scorer treats parallel tool calls (BFCL's parallel category) as one step with N tool calls; the definitional note from chapter 04 applies.

### ToolBench (ToolLLM)

**Benchmark.** Qin et al., "ToolLLM: Facilitating LLMs to Master 16000+ Real-World APIs" (<https://arxiv.org/abs/2307.16789>, 2023). A dataset built from RapidAPI with ~16,000 real APIs and ~469 toolsets, plus an evaluator ("ToolEval") that uses LLM-as-judge for *pass rate* and *win rate* against reference trajectories.

**Shape to borrow.** The **tool-selection-under-many-tools** scenario — the agent is given a large pool and must pick the right subset. If your product has a long tool catalog, the ToolBench pattern is a useful stress-test template: subsample your own tool catalog to ~50 tools around the task family and score tool selection against the reference.

**Shape to be careful with.** ToolBench's evaluator is LLM-as-judge. Judge-based scoring is appropriate for free-form arguments (chapter 03 Level 5) and risky for anything else; the chapter-02 argument matcher only promotes a call to Level 5 after Level 1–4 fail. Do not borrow ToolBench's judge choice uncritically.

**Where it plugs in.** The chapter-02 "unauthorised_tool" verdict and chapter-05's "keep the authorised set explicit" rule are the companion mechanics for a long-tool-catalog setup; ToolBench's shape stresses them.

### A unified mapping table

The seven benchmarks and where each one plugs into this module's scorer stack:

| Benchmark | Borrow | Replace | Plugs into |
|---|---|---|---|
| SWE-bench Verified | Programmatic verifier (apply+test+exit) | Open-source repos → your repos | Ch-03 "programmatic verifier"; Ch-05 Scorer |
| τ-bench | User simulator; `pass^k`; DB-state scorer | Airline/retail domains → yours | Ch-05 Solver; Ch-04 budget on simulator; Ch-03 state scorer |
| WebArena | Programmatic end-state check for browsing | 5 self-hosted sites → your UI | Ch-05 Solver (browser); Ch-02 DAG; Ch-03 programmatic |
| GAIA | Difficulty-tier taxonomy | 466 questions → your samples | Ch-05 dataset stratification |
| AgentBench | One scorer family per environment | 8 environments → your task families | Ch-05 one Task per task family |
| BFCL | AST evaluator; function-calling taxonomy | BFCL tool catalog → yours | Ch-02 Level-4 arg matcher |
| ToolBench | Tool-selection-under-many-tools scenario | 16k RapidAPI → your catalog | Ch-02 unauthorised_tool; Ch-05 tool pool |

The table is deliberately opinionated: each row says *one* thing to borrow and *one* thing to leave on the leaderboard. In practice some surfaces will want to borrow more; the point is to be explicit about which parts.

### How to run a shape-template adoption

A compact process the exercise at the end of the module uses:

1. **Pick the benchmark whose shape most closely matches your task family.** Not the benchmark whose score is highest for your model; the one whose *shape* matches.
2. **Enumerate what you will keep.** Task schema? Scoring function? Harness integration? Metric name? Record each as a bullet.
3. **Enumerate what you will replace.** Dataset? Tools? Reference trajectories? Each gets a bullet.
4. **Build a one-page shape-template plan.** The plan goes next to the eval plan (mod-101). It names the public benchmark, the module in this track that owns the implementation (`mod-103` for trajectory scoring, `mod-105` for RAG triad, `mod-108` for safety), and the fixture-hash + version discipline (mod-106).
5. **Score five historical trajectories.** Shadow the shape-template scorer against five past trajectories whose outcomes you know. If the scorer disagrees with your prior labels, adjudicate; if it agrees, roll it into the regression suite.

The exercise `exercise-04-public-benchmark-shape-template-audit.md` walks the audit end to end for the task families on the module's running support-agent surface.

## Example — τ-bench shape, support-agent content

The support-agent surface has multi-turn interactions, tool calls that mutate state (`open_ticket`), a database that stores the outcome, and sampling noise sensitivity (the model is non-zero-temperature). τ-bench's shape fits.

What is borrowed:
- The **user simulator** is a second LLM that plays the customer. The simulator reads a scenario card ("your order 12345 has not shipped; you want to know why") and emits the customer's next turn conditioned on the agent's last reply.
- The **database-state scorer** reads the CRM and the ticket store at the end of the dialogue and compares to the reference: `ticket_opened=true, reason="no_shipping_update", order_id=12345`.
- The **`pass^k` reporting**: run each scenario `k=5` times at the production temperature; report the fraction of scenarios where **all five** runs reached the correct end state.

What is replaced:
- The scenario distribution. 100 scenarios drawn from the support-agent's own task families (not τ-bench's airline/retail), stratified by the GAIA-shaped difficulty tiers (Level 1: one tool, Level 2: two tools with a dependency, Level 3: three or more tools and policy branching).
- The toolset. `docs.search`, `crm.lookup`, `open_ticket` — the surface's own tools; not τ-bench's reservation and refund tools.

What is left on the leaderboard:
- τ-bench's own airline/retail scores. The team's "we're at 62% on τ-bench-airline" would be nice to know for shop-talk purposes, but it is not the number the rollout gate reads.

The resulting Task is implemented as an Inspect Task (chapter 05) with a `Solver` that uses the simulator, a Scorer stack that includes the chapter-02 per-step scorer, chapter-03 rubric, chapter-04 budget scorer, and a τ-shape **database-state scorer** that reads the CRM and ticket store. The `pass^k` roll-up is a wrapper around `inspect eval` that invokes the Task `k` times per scenario.

## Common pitfalls

- **Borrowing the number instead of the shape.** "We're at 62% on τ-bench" is a model-capability sentence, not a product-eval sentence. Do not route a rollout decision through a leaderboard score.
- **Adopting the dataset wholesale.** Public benchmarks' datasets are not your traffic. Replace the inputs; keep the schema.
- **Importing contamination risk.** A benchmark the frontier models have memorised scores artificially high. Your product's eval has to be on data that cannot be memorised — your own traffic, versioned in mod-110, retention-controlled per mod-102 chapter 06.
- **Letting the benchmark's judge pick your judge.** ToolBench and some τ-bench variants use LLM-as-judge; your product's judge configuration is a mod-104 decision. Pick your judge; use the benchmark's shape.
- **Over-stacking shapes.** Borrowing from six benchmarks for one task family is cargo-culting. One or two per family, each with a clear purpose, is enough.
- **Hiding adoption behind "we run τ-bench nightly".** The scorer has to be legibly *your product's scorer with τ-bench's shape*, not "τ-bench". A reviewer looking at the dashboard should see your task families, not the airline / retail names.

## Summary

- Public agent benchmarks are **shape templates**: task schema, scoring function, harness, reporting metric. Borrow the shape; replace the content.
- SWE-bench Verified: programmatic verifier (apply + test + exit). Borrow the pattern for code-fix agents on your own repos.
- τ-bench: user simulator, DB-state scorer, `pass^k` reliability metric. Borrow for multi-turn, state-mutating agents.
- WebArena: programmatic end-state scoring for browsing agents. Borrow for UI-navigation agents on your own UI.
- GAIA: difficulty-tier taxonomy. Borrow for stratified reporting; do not borrow its final-answer-only scoring.
- AgentBench: one scorer family per environment. Maps onto one Task per task family in Inspect.
- BFCL: AST-based function-call evaluator; function-calling taxonomy. Borrow the AST matcher for code-shaped arguments.
- ToolBench: tool-selection under a large catalog. Borrow the stress-test scenario; do not inherit the LLM-judge metric.
- Run an adoption as a one-page plan: borrow X, replace Y, mod-110 owns the fixture, mod-106 owns the hash discipline, score five historical trajectories to validate before going into CI.
- The leaderboard number is not a product SLO. Avoid conflating the two in dashboards, memos, and rollout decisions.

Chapter 07 defines deterministic replay for failing trajectories — the post-mortem workflow that completes the scorer / budget / replay triad.
