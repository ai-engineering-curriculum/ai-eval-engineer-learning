# quiz-01: Product-Shaped Evaluation Foundations

**Estimated time:** 15–20 minutes
**Covers:** chapters 01–06 of this module.

Answer each question, then check the key at the bottom. Explanations cite the chapter the answer comes from so you can re-read the relevant section if an answer surprised you.

---

## Questions

### 1. What counts as a "surface" in this track?

- A. The underlying frontier model that several product features share.
- B. The product as a whole, e.g. "our chatbot."
- C. A specific user-visible interaction — e.g. the *summarise this thread* action inside the inbox — together with its prompt, retriever, tools, guardrails, decoding, and SLO.
- D. A single API endpoint exposed by the inference gateway.

### 2. The same model backs three features: an inbox summariser, a sales email drafter with a CRM tool call, and a support RAG-plus-ticketing agent. How many eval plans does the module say you should write?

- A. One, keyed off the shared model version.
- B. One per feature team.
- C. Three — one per surface, because prompts, retrievers, tools, and SLOs differ even though the model is shared.
- D. Two — one for the chat-shaped surfaces and one for the tool-calling surface.

### 3. A colleague writes: "Our SLA is that 95% of summaries are faithful." Which vocabulary rule from chapter 01 does that statement violate?

- A. It is missing a *window* on the objective.
- B. It conflates SLA (a customer contract with a remedy) with SLO (an internal target on an SLI) — this track authors SLIs and proposes SLOs; the SLA is a product/legal artefact that consumes them.
- C. SLAs apply only to latency, never to quality.
- D. The number should be expressed as an odds ratio, not a percentage.

### 4. When you first read a product spec, chapter 02 tells you to tag it for four kinds of information. Which set matches?

- A. Functional, non-functional, performance, security.
- B. Intent claims, grounding claims, constraint claims, distribution claims.
- C. Happy path, unhappy path, edge case, failure case.
- D. User stories, acceptance criteria, risks, dependencies.

### 5. Chapter 02 insists every criterion must be *falsifiable* and have a *numeric* SLI. Which proposed row violates both rules?

- A. "Share of judged summaries with faithfulness score ≥ 4 on a 1–5 rubric, ≥ 92% weekly on sampled English traffic."
- B. "Summaries are high quality and users are happy."
- C. "p95 end-to-end latency < 60s on sampled live traffic."
- D. "0 leaks on the HR-seeded fixture pre-merge."

### 6. The "so what" test from chapter 02 has two parts. Which pair is correct?

- A. Reproducibility and auditability.
- B. Actionability ("if this SLI drifts, what does the team do?") and attribution ("if this SLI moves, can we tell which change caused it?").
- C. Cost and latency.
- D. Coverage and sampling.

### 7. Chapter 03 gives a four-step decision rule for placing a criterion. In what order do you walk it?

- A. Offline → Online → Deferred → Both.
- B. Deferred (needs peer methodology depth?) → Offline (must a change pass/fail on it?) → Online (only meaningful on live distribution?) → Both (a coarse offline gate plus a finer online SLI).
- C. Cheapest-first — rule checks, then judges, then humans.
- D. Online → Offline → Both → Deferred.

### 8. Which of the following is chapter 03's own list of what *offline* regression suites are good at?

- A. Drift detection, novel-jailbreak discovery, cost-under-real-load.
- B. Regression protection, attribution, ship gates, reproducibility.
- C. Measuring the tail of the live-user distribution.
- D. Replacing human review entirely.

### 9. Chapter 03 names one thing online eval is specifically *bad* at. Which?

- A. Measuring cost and latency under real load.
- B. Detecting drift.
- C. Gating a merge, because online SLIs are lagging by definition.
- D. Catching novel failures that offline fixtures could not enumerate.

### 10. Chapter 04 describes how this track uses ISO/IEC 25059. Which usage is correct?

- A. As the authoritative source of SLO numbers for each surface.
- B. As a per-surface completeness checklist — every characteristic is answered `in scope`, `deferred to a named peer`, or `not applicable with a one-sentence reason`.
- C. As a replacement for the NIST AI RMF.
- D. As a certification you must pass before shipping.

### 11. Where does an eval plan this track authors sit inside the NIST AI RMF?

- A. It is a Govern-function artefact — it sets policy and accountability.
- B. It is a Manage-function artefact — it makes the risk-acceptance decision.
- C. It is a Measure-function artefact; Govern and Manage belong to the Governance peer track, and Map is a joint activity.
- D. The RMF does not apply to application-altitude evaluation.

### 12. A stakeholder forwards an MT-Bench ranking as justification for swapping the model behind a customer-support triage surface. Chapter 05 lists eight questions to walk with them. Which outcome is the chapter's *most common* classification?

- A. Accept as-is — most popular public benchmarks transfer well.
- B. Reject without a replacement proposal.
- C. Adapt — use the benchmark as a *shape template* for an internal suite built against the surface's real distribution, scaffold, and metric needs.
- D. Ignore the request until the surface has an incident.

### 13. Which failure mode from chapter 05 best explains why "model A ranks above model B at 87% vs 85% aggregate" can still lead to a worse user experience on model A?

- A. Prompt sensitivity.
- B. Aggregate-hides-the-tail — users experience the 13% tail, not the 87% average, and two models at the same aggregate can have very different tail behaviour.
- C. Judge-model shift.
- D. Static benchmark ageing.

### 14. Chapter 06 lists four anti-patterns for the two-page plan. Which is *not* on that list?

- A. The "we will measure quality" plan — no families, no SLIs, no gates.
- B. The "kitchen sink" plan — 40 criteria, none prioritised, all deferred.
- C. The "silent deferral" plan — half the criteria are quietly not addressed.
- D. The "version-pinned" plan — the plan names the exact model, prompt, retriever, decoding, and fixture-hash versions it applies to.

### 15. Chapter 06 requires every deferred criterion to name a peer owner and an interface. Which of the following is the correct peer for *full judge-vs-human calibration methodology* (κ, Spearman, bootstrap CIs at methodology depth)?

- A. `ai-risk-engineer`.
- B. `ai-evaluation-engineer` (Governance family, level 25).
- C. `model-evaluation-engineer` (peer, level 30, ML Engineering family).
- D. `ai-infra-security`.

---

## Answer key

1. **C.** Chapter 01 defines a surface as a specific user-visible interaction with its own intent contract, runtime shape, config, SLO, and owner; "our chatbot" is explicitly called out as *not* a surface.
2. **C.** Chapter 01's three-surfaces example shows that sharing a model does not imply sharing a plan — different prompts, retrievers, tools, and SLOs mean each surface deserves its own plan.
3. **B.** Chapter 01 reuses the Google SRE vocabulary verbatim: SLI (indicator), SLO (objective), SLA (customer contract with a remedy). The track authors SLIs and proposes SLOs; SLAs are a product/legal decision.
4. **B.** Chapter 02 names the four tags explicitly: intent claims, grounding claims, constraint claims (safety/cost/latency), and distribution claims. Anything else risks being a criterion the spec never actually implied.
5. **B.** Chapter 02's "every criterion must be falsifiable by a procedure and have a numeric SLI" rule fails on "summaries are high quality and users are happy" — no procedure, no number.
6. **B.** Chapter 02's "so what" test is actionability plus attribution; both must pass before a criterion goes into the plan.
7. **B.** Chapter 03 gives the rule as: defer-if-peer-owns-the-depth → offline if a change under review must pass/fail → online if only meaningful on live distribution → both (the common answer).
8. **B.** Chapter 03 names regression protection, attribution, ship gates, and reproducibility as offline's strengths; drift and live-distribution SLIs belong to online.
9. **C.** Chapter 03 lists "ship gates" explicitly in the "what online is bad at" section — online SLIs are lagging by construction.
10. **B.** Chapter 04 describes 25059 as a per-surface completeness checklist with the three allowed row states; it does not pick SLO numbers and is not a certification.
11. **C.** Chapter 04 states the plan is a Measure-function artefact; Govern and Manage are the Governance peer track's scope, and Map is a joint activity.
12. **C.** Chapter 05's "shape template" pattern is the most common outcome: adopt the task shape but replace inputs, references, and scaffold to match the surface. Pure accepts are rare; pure rejects happen when distribution or contamination fails hard.
13. **B.** Chapter 05 lists "aggregate hides the tail" as a load-bearing failure mode — users experience the tail, and two models at the same aggregate can diverge there.
14. **D.** Chapter 06 lists exactly four anti-patterns: "we will measure quality," "kitchen sink," "silent deferral," and "benchmark alignment." Version-pinning is required by section 3 of the template, not an anti-pattern.
15. **C.** Chapters 01, 03, and 06 all point judge-vs-human calibration methodology (κ, Spearman, bootstrap CIs at methodology depth) at the peer `model-evaluation-engineer` (level 30, ML Engineering family).
