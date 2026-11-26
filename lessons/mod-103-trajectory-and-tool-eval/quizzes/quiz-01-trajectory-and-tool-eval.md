# quiz-01: Trajectory and Tool-Call Evaluation for Production Agents

**Estimated time:** 15–20 minutes
**Covers:** chapters 01–07 of this module.

Answer each question, then check the key at the bottom. Explanations cite the chapter the answer comes from so you can re-read the relevant section if an answer surprised you.

---

## Questions

### 1. Chapter 01 states the five questions every trajectory scorer in this module must answer "on the trajectory record alone". Which of the following is *not* one of them?

- A. Did the agent take the right steps (name, arguments, ordering with dependencies)?
- B. Did the agent land on the right final answer against the reference?
- C. Did the agent do it inside the allowed budgets (cost, latency, steps, retries)?
- D. Did the model used at this step beat its public-benchmark leaderboard rank this week?

### 2. Chapter 01's running example — the support agent that answered `"Where is my order 12345?"` by calling `sql_query` instead of `docs.search` — is the chapter's exemplar for which failure mode that trajectory eval catches but end-state eval misses?

- A. Right-tool / wrong-order (dependency violated).
- B. Right-answer / wrong-tool (golden shortcut).
- C. Wrong-final / catastrophic budget.
- D. All three are the same failure mode under different names.

### 3. Chapter 02 lists four forms of reference trajectory. Which one does the chapter name as the production default for multi-step agents with real data dependencies?

- A. **Strict** — the exact ordered sequence of `(tool_name, arguments)` tuples.
- B. **Set** — a multiset of tool calls with flexible order.
- C. **DAG** — a partial order expressed as a directed acyclic graph of prerequisites; any topological sort is valid.
- D. **Programmatic** — a Python predicate that decides acceptability at scoring time.

### 4. Two scoring calls arrive for the same `docs.search` step: `expected = {"query": "status of order 12345", "top_k": 3}`, `actual = {"top_k": 3, "query": "order 12345 status"}`. Chapter 02 lays out a five-level argument-match ladder. Which pairing of levels matches the chapter's walked example?

- A. Level 1 structural: match; Level 2 exact-value after canonicalisation: no match; Level 3 semantic-equivalence with a token-set rule: match.
- B. Level 1 structural: no match (keys are in a different order); Level 2 exact-value: match once sorted.
- C. All five levels agree on "match" for this call because the keys and types are identical.
- D. Level 4 AST is the only level that resolves this, because the arguments are text.

### 5. Chapter 02's ordering matchers include a partial-order (topological) matcher that uses Python's `graphlib.TopologicalSorter`. The reference DAG has `docs.search → open_ticket` and `crm.lookup → send_notification`. Which observed sequence does the chapter cite as **invalid** (dependency violated)?

- A. `crm.lookup → docs.search → send_notification → open_ticket`
- B. `docs.search → open_ticket → crm.lookup → send_notification`
- C. `open_ticket → docs.search → crm.lookup → send_notification`
- D. `docs.search → crm.lookup → send_notification → open_ticket`

### 6. Chapter 02's name matcher lists four adjustments on top of exact-string equality. Which of the following is **not** one of them (and is instead handled by chapter 04's budget scorer)?

- A. Alias table for tools renamed between model versions (`docs.search` → `docs.retrieve`).
- B. Namespaced MCP-style prefixes scored as `namespace + name` as one unit.
- C. Consecutive identical `(tool_name, arguments)` retries collapsed before comparison, with retry **count** retained for the budget scorer.
- D. NFKC + casefold normalisation of `tool.name` before comparison.

### 7. Chapter 03 lists six methods for scoring the final-answer step. Which of the following is the chapter's framing of the **programmatic verifier** method, and which public benchmark does the chapter cite as the paradigm case?

- A. "Normalise and string-compare to the reference"; cited paradigm: GAIA.
- B. "Apply the candidate output under a rule the task family's spec defines (regex, JSON Schema, test suite) and read the exit status"; cited paradigm: SWE-bench Verified.
- C. "An LLM judge grades the reply against a rubric prompt"; cited paradigm: ToolBench.
- D. "Classic NLP overlap (BLEU / ROUGE / token F1)"; cited paradigm: τ-bench.

### 8. Chapter 03 separates `answer_credit` from `groundedness_credit` as two distinct rubric axes. Which trajectory does the chapter name as the canonical case that this separation is designed to catch?

- A. A reply whose content is **wrong** but was drawn from the authoritative retriever.
- B. A reply whose content is **correct** but was drawn from an unauthorised source (the chapter-01 ETA that came from `sql_query` instead of the `docs.search` policy doc).
- C. A reply that is blocked by the output guardrail before it reaches the user.
- D. A reply that comes back empty because `max_messages` was hit.

### 9. Chapter 03 states three design rules that keep a weighted-sum rubric honest. Which triple matches the chapter?

- A. (a) re-tune weights until the sample's pass-rate is above 0.95; (b) one rubric across all task families to keep comparisons fair; (c) emit only the scalar score.
- B. (a) pre-register weights per task family; (b) impose per-verdict **floors** so a wrong-tool step cannot be averaged away; (c) preserve the dimensional `axes` + `flagged_verdicts` breakdown alongside the scalar.
- C. (a) hide `flagged_verdicts` from reviewers; (b) use mean-score aggregation; (c) tune per-cohort thresholds post-hoc.
- D. (a) let an LLM judge grade the whole trajectory; (b) use a soft quadratic penalty for budgets; (c) carry only the mean score across trajectories.

### 10. Chapter 04 defines `budget_credit` with a specific aggregation across the four budget families. Which formula matches the chapter?

- A. Weighted mean across families with weights summing to 1.0.
- B. `budget_credit = min over families of family_credit`, where `family_credit = clamp(1 - over_fraction, 0, 1)` and `over_fraction = max(0, observed/ceiling − 1)`. Any family at 2× its ceiling pins the family credit (and thus `budget_credit`) at 0.
- C. Quadratic penalty `1 − (over_fraction)²` summed across families.
- D. Boolean AND over per-family pass/fail with no partial credit.

### 11. Chapter 04 makes two definitional notes to prevent "ghost retries" and miscounted steps in the budget dashboard. Which pair matches the chapter?

- A. A step with parallel tool calls counts as **N** steps and **N** tool calls; a collapsed retry counts as an additional tool call each time it fires.
- B. A step with parallel tool calls is **one** step and **N** tool calls; a collapsed retry is **one** tool call plus a retry count (the retry is a harness event, not a plan decision).
- C. A step with parallel tool calls is one tool call; retries are user-visible re-asks and so count as additional tool calls.
- D. Steps, tool calls, and retries are interchangeable integers; pick whichever the dashboard shows.

### 12. Chapter 04 insists on an **attribution block** next to the per-family credits. Which three fields does the chapter name, and why?

- A. `wrongest_step`, `most_expensive_prompt`, `highest_judge_disagreement` — so the judge-grader can be re-trained.
- B. `cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool` — so a rollout gate that says "cost over budget" points at a specific span and tool, turning a bug report into a stack trace.
- C. `task_family`, `rubric_version`, `price_table_version` — so historical runs are comparable.
- D. `floor_tripped`, `pass_threshold`, `pass_rate` — so the on-call can read pass-rate off the dashboard.

### 13. Chapter 05 adopts **Inspect** (UK AISI) as the module's reference harness and names five primitives. Which set matches the chapter?

- A. `Agent`, `Session`, `Turn`, `Reward`, `Transition`.
- B. `Task`, `Sample`, `Solver`, `Scorer`, `Tool`.
- C. `Pipeline`, `Step`, `Dataset`, `Metric`, `Sandbox`.
- D. `Experiment`, `Variant`, `Judge`, `Gate`, `Rollout`.

### 14. Chapter 05 maps this module's scorers onto a Scorer stack inside one Inspect `Task` — per-step, final-answer, budget, and a **rubric** scorer at the end. Which three conventions does the chapter cite for keeping the stack composable?

- A. Each scorer is pure (reads `TaskState`, does not mutate); the rubric scorer runs last and is the only one that sets `Score.value`; the full per-axis breakdown lives in `Score.metadata`.
- B. One monolithic scorer per Task; mutate `TaskState` between scorers; return only a scalar.
- C. Each scorer owns its own dataset slice and judges only its own samples.
- D. Scorers re-execute the agent to double-check the trajectory.

### 15. Chapter 05 lists three things that live **outside** Inspect even though it is tempting to pull them in. Which triple matches?

- A. The dataset / fixture lifecycle (owned by mod-110); the judge configuration (owned by mod-104); the rollout gates (owned by mod-106 and mod-107).
- B. The Solver, the Scorer stack, and the Tool catalog.
- C. The model identifier, the `message_limit`, and the `time_limit`.
- D. The eval log, the log viewer, and the OpenInference auto-instrumentor.

### 16. Chapter 06 frames seven public benchmarks (SWE-bench Verified, τ-bench, WebArena, GAIA, AgentBench, BFCL, ToolBench) as *shape templates*. Which pairing of benchmark → one-thing-to-borrow does the chapter make?

- A. SWE-bench Verified → the exact 2,294-issue dataset; τ-bench → the airline and retail scenarios; GAIA → the leaderboard number.
- B. SWE-bench Verified → the programmatic verifier (apply + run tests + exit status); τ-bench → the user-simulator + DB-state scorer + `pass^k` reliability metric; BFCL → the AST-based function-call evaluator for code-shaped arguments.
- C. WebArena → use the five self-hosted sites unchanged; GAIA → adopt the exact-match-on-short-answer scoring as your product scorer.
- D. ToolBench → inherit its LLM-as-judge for free-form arguments by default; AgentBench → collapse all task families into one Task.

### 17. Chapter 06 names what you should *not* reuse from a public benchmark. Which of the following is the chapter's framing of the biggest misuse?

- A. Adopting the schema for the task input.
- B. Reusing the metric name (`pass^k`, `pass@1`) in a cross-team conversation.
- C. Routing a rollout decision through the leaderboard number — "we're at 62% on τ-bench-airline" is a model-capability sentence, not a product-eval sentence.
- D. Running the public benchmark's harness end-to-end once as a sanity check.

### 18. Chapter 07 enumerates the **six pins** that must be fixed to achieve deterministic replay. Which set matches the chapter?

- A. (1) model identity (served, not requested — `gen_ai.response.model`); (2) sampling parameters (temperature, top_p, seed, etc.); (3) the exact message list sent at each step; (4) tool returns (via a recorded tool-response table); (5) environment (clock, feature flags, locale, agent id); (6) harness version (Inspect, Python, SDK).
- B. Model identity, user id, prompt template, retrieval index, cost ceiling, rubric version.
- C. The eval plan, the rubric, the fixtures, the tools, the judges, the dashboards.
- D. CPU, GPU, OS, container image, network, DNS.

### 19. Chapter 07 describes the **recorded tool-response table** — the central replay artefact. Which design rule does the chapter name for tools that were retried on the original run?

- A. Keep only the final successful return and discard the earlier failed attempts, to reduce storage.
- B. Record an ordered `return_sequence` (e.g., two 502s followed by the OK) so the replay harness yields each entry in order and the retry storm reproduces deterministically.
- C. Re-execute the real tool on replay to observe current behaviour.
- D. Store the retries as a set (order not preserved) since the model should not depend on order.

### 20. Chapter 07 describes a six-step post-mortem workflow that uses the replay harness. Which ordered sequence matches?

- A. Bisect → Lift → Pin → Reproduce → Fix → Learn.
- B. Lift → Pin → Reproduce → Bisect (change one pin at a time) → Fix → Learn (promote the fixture into the regression set).
- C. Reproduce → Fix → Lift → Pin → Learn → Bisect.
- D. Fix → Learn → Lift → Pin → Reproduce → Bisect.

### 21. Chapter 07 names four cases where **exact** replay is impossible. Which of the following is **not** one of them?

- A. The exact `gen_ai.response.model` has been retired by the provider (fall back to approximate replay flagged `model_substituted=true`).
- B. The trajectory included a side-effectful step (`pay_invoice`) whose "return" was dependent on live downstream state.
- C. A Collector-side PII scrubber redacted tool payloads before the replay-eligible retention tier captured them.
- D. The scorer's rubric weights have been updated since the trajectory ran (just score under the new weights).

---

## Answer key

1. **D.** Chapter 01's "five trajectory questions" are: (1) right steps (name/args/ordering), (2) right final answer, (3) inside budgets, (4) replayable deterministically, (5) partial-credit when partly right. Leaderboard rank is explicitly called out elsewhere as a shape-template signal, not a trajectory-scorer question.
2. **B.** Chapter 01's "three failure modes" section labels the `sql_query` substitution as the **golden-shortcut** / right-answer, wrong-tool case — the ambient-permissions blast radius a final-answer judge misses.
3. **C.** Chapter 02's reference-trajectory table says DAG is the production default for multi-step agents with real data dependencies; Programmatic is the escape hatch for the ~10–20% of tasks whose correct trajectory branches on tool returns. Strict and Set are called out as bad defaults.
4. **A.** Chapter 02's worked `docs.search` example: Level 1 structural passes (same keys/types), Level 2 exact-value after canonicalisation fails (`"status of order 12345"` ≠ `"order 12345 status"`), Level 3 semantic-equivalence with a token-set rule passes. The chapter uses this exact example to motivate the ladder.
5. **C.** Chapter 02's worked ASCII example: `open_ticket → docs.search → crm.lookup → send_notification` violates the edge `docs.search → open_ticket` (open_ticket fires before docs.search). The chapter cites this as the invalid interleaving.
6. **C.** Chapter 02 is explicit that retries are a **harness** event, not an agent decision. The name matcher collapses consecutive identical `(tool_name, arguments)` calls before comparing to the reference; retention of the retry **count** is chapter 04's budget-scorer concern.
7. **B.** Chapter 03's "programmatic verifiers" row cites SWE-bench (apply the patch, run the project's test suite, read the exit status) as the paradigm case. Chapter 06 then borrows the shape for code-fix agents on your own repos.
8. **B.** Chapter 03's "Groundedness as a separate axis" section cites chapter 01's ETA-from-`sql_query` case as the canonical right-content / wrong-source trajectory the split is designed to catch. `answer_credit=1.0, groundedness_credit=0.3` is the chapter's illustrative outcome.
9. **B.** Chapter 03's three design rules are: pre-register weights per task family, impose per-verdict **floors** so a weighted sum alone cannot average a `wrong_tool` away, and preserve the dimensional breakdown + `flagged_verdicts` next to the scalar. The other options pair post-hoc tuning, one-rubric-across-families, or hiding the breakdown — all named as pitfalls.
10. **B.** Chapter 04's "The composite budget score" states exactly this formula: linear decay to zero per family, worst-family-wins via `min` across families, pinned at 0 at 2× over any ceiling. The chapter explicitly rejects weighted-mean and quadratic alternatives.
11. **B.** Chapter 04's "Step count, tool count, retry count" section: a step is one iteration of the agent loop (parallel calls still = one step); a collapsed retry is one tool call plus a retry count; the alternative definitions the other options list are called out as the sources of ghost retries.
12. **B.** Chapter 04's "Attribution and the 'find the span' rule" names `cost_dominant_span`, `latency_dominant_span`, and `retry_dominant_tool`; the chapter's framing is that a verdict without an attribution block is "a bug report with no stack trace".
13. **B.** Chapter 05 opens with Inspect's five primitives: `Task` (benchmark unit), `Sample` (one input), `Solver` (drives the agent), `Scorer` (reads terminal state), `Tool` (decorated function the agent can call).
14. **A.** Chapter 05's "Mapping the module's scorers" section states exactly this triple: scorers are pure, the rubric scorer runs last (only it sets `Score.value`), and the per-axis breakdown lives in `Score.metadata`. A monolithic scorer is called out as a pitfall.
15. **A.** Chapter 05's "What *not* to put in Inspect": the dataset owner (mod-110), the judge model configuration (mod-104), and the rollout gate (mod-106 / mod-107). Each is owned by a downstream module.
16. **B.** Chapter 06's unified mapping table: SWE-bench Verified → programmatic apply+test+exit verifier; τ-bench → user simulator + DB-state scorer + `pass^k`; BFCL → AST evaluator for code-shaped arguments. The other options are explicitly called out as misuses in the "Common pitfalls" section.
17. **C.** Chapter 06's "Common pitfalls" leads with "Borrowing the number instead of the shape" and uses the "we're at 62% on τ-bench" example verbatim — a model-capability sentence, not a product-eval sentence. The leaderboard number is the one shape-reuse that is called out as always wrong.
18. **A.** Chapter 07's "What has to be fixed for a replay" lists exactly these six pins and names `gen_ai.response.model` (not `request.model`) as the right attribute for model identity.
19. **B.** Chapter 07's "The recorded tool-response table" explicitly says: record the full retry sequence; the replay harness yields each entry in order so the retry storm reproduces. Collapsing to the final return is called out as losing the retry-storm attribution.
20. **B.** Chapter 07's "The post-mortem workflow" lists the six steps in this exact order: **lift, pin, reproduce, bisect, fix, learn** — and names each experiment as varying **one** pin at a time.
21. **D.** Chapter 07 names four impossibility cases: model retirement, side-effect-in-the-loop, uncaptured external state, and PII scrubbing that destroyed the pin. A rubric-weights change is a mod-110 / mod-106 concern — you simply re-score under the new rubric; the trajectory itself is still replayable.
