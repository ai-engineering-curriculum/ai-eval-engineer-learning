# Build-vs-Buy Eval Platform Matrix: The Decision Refreshed on a Cadence

## Motivation

Chapter 01 named the *platform churn* failure shape: the eval platform is rewritten every 18 months because the last decision was made informally, and each rewrite loses history, invalidates old dashboards, and takes six months to rebuild parity. The rewrite-cycle burns the eval program's actual time — the release-gate policy stops being maintained, the delegation contracts drift, and the incident investment loop stalls, all because the team is busy migrating platforms.

The fix is not "pick the right platform once." The fix is a **build-vs-buy matrix** — the eval platform stack decomposed into named components, each component's current decision (buy / build / hybrid) recorded with its evidence and next-refresh date, and the whole matrix revisited on an annual cadence with quarterly status checks. The matrix is the artefact that survives the specific vendor going out of business, the specific OSS tool losing its maintainer, and the specific pricing model changing — because the *shape of the decision* is stable even when the current answer moves.

Two symptoms show the matrix is missing:

- **The platform question re-opens every product review.** "Should we still be on Langfuse?" comes up at every quarterly because nobody has a record of *why* Langfuse was picked over Phoenix in the first place, or what would trigger a swap. The re-litigation burns team cycles and never resolves.
- **The platform is picked by whoever set it up first.** Someone stood up Phoenix as a proof-of-concept two years ago; it is still the store because "we're already on it." No one has compared it to alternatives against actual usage; the sunk-cost gravity is the only reason it is still the answer.

Both symptoms are decision-shape failures. This chapter closes them.

The chapter is deliberately vendor-neutral in its recommendations. Which specific platform is right for a specific org depends on org-specific data-residency, extensibility, cost, and existing-infra constraints. What is *not* org-specific is the *shape* of the decision — the component decomposition, the evaluation axes, and the refresh cadence.

## Core concepts

### Decompose the platform into components, not one big buy

The single biggest mistake in eval-platform selection is treating "the platform" as a monolith and picking one product to be the answer. Real eval platforms are stacks; different components have different economics, different vendor landscapes, and different swap costs. Decomposition is the first move.

The mod-110 platform slice named seven load-bearing tables (`traces`, `spans`, `eval_sets`, `eval_runs`, `eval_results`, `rubrics`, `pricing_snapshots`); the platform-stack decomposition at chapter altitude names *services*, not tables. A reasonable decomposition:

| # | Component | What it does | Common OSS / vendor options (non-exhaustive; verify current) |
|---|---|---|---|
| 1 | Trace collection | OTel-GenAI spans emitted from the app; ingestion into a store | OpenTelemetry SDKs; Phoenix collector; Langfuse SDK; Weave (W&B) client; Braintrust logs; provider SDK auto-instrumentation |
| 2 | Trace storage & query | Backing store for traces + spans + attributes; queryable by lineage | Phoenix (Arize); Langfuse; Braintrust; Weave; Humanloop; Galileo; Patronus; self-hosted OTel + ClickHouse / DuckDB |
| 3 | Eval set / test case management | Versioned dataset store, human-review workflows, dataset lineage | Braintrust datasets; Langfuse datasets; Humanloop datasets; Phoenix datasets; Argilla; DVC + Git; internal Postgres |
| 4 | Rubric / judge runner | Executes rubrics (LLM-as-judge or programmatic) against outputs | Promptfoo; DeepEval; RAGAS; Braintrust `Eval()`; Phoenix `evaluators`; Langfuse; internal harness |
| 5 | Offline gate CI integration | Binds the mod-106 replay bundle to the CI/CD pipeline | Promptfoo GH Actions; Braintrust CI; DeepEval CI; internal GitHub Actions / GitLab CI shim |
| 6 | Online eval / monitoring loop | Continuous scoring on production traffic; drift detection; alerts | Langfuse; Phoenix; Arize; Galileo; Patronus; Datadog LLM Observability; New Relic AI Monitoring; internal |
| 7 | Safety / guardrail infrastructure | Runs adversarial eval; scores against hazard taxonomies | Llama Guard; NeMo Guardrails; Guardrails AI; Presidio; OpenAI Moderation; Azure AI Content Safety; garak; PyRIT |
| 8 | Human-review / annotation | Reviewer UI, adjudication, disagreement resolution | Argilla; Braintrust human review; Humanloop; Label Studio; internal |
| 9 | Cost & pricing accounting | Per-request cost with vendor pricing, cache, batch accounting | Vendor consoles + internal shim; Helicone; Langfuse; Braintrust |
| 10 | Cloud-native provider eval | Vendor-managed eval offering for a specific cloud | Google Vertex AI Evaluation; AWS Bedrock Evaluations; Azure AI Foundry Evaluations |

<!-- needs-research: on the next research cycle, re-verify the platform / vendor list above against current offerings — the app-eval landscape re-shuffles roughly quarterly (product renames, new SKUs, deprecations); update names and confirm the components each vendor covers today. -->

Ten components. Each has its own build-vs-buy decision, its own evaluation axes, and its own refresh interval. A team that treats "the platform" as one buy is not decomposing.

### The evaluation axes — six, applied to every component

For each component the matrix asks the same six questions. Consistency across components matters for the same reason consistency across gate policies and delegation contracts matters — the reader can walk the matrix without relearning the shape.

| # | Axis | What it asks | Why it matters |
|---|---|---|---|
| 1 | **Cost of ownership** | List price + integration cost + operating cost + swap-out cost. Include the eval team's time to operate the tool. | The listed price is rarely the total cost; a "free" OSS with three engineers operating it is not free. |
| 2 | **Data residency and sovereignty** | Where does the data land (region, cloud); what data crosses the boundary; what contract governs data use for training / improvement | For regulated surfaces (EU AI Act high-risk, medical, financial) the residency answer is often the deciding constraint. |
| 3 | **Extensibility** | Can the team add a custom rubric / evaluator / trace attribute / storage type without a vendor-side change; is the extension supported or hack-and-hope | A custom rubric that the vendor doesn't natively support is either an extension surface or a permanent shadow-system. |
| 4 | **Vendor lock-in** | How reversible is a swap-out; is the schema portable; do the historical artefacts (traces, eval-set versions, rubric definitions) come with you | The 18-month rewrite pain is entirely a lock-in symptom; the axis measures the pain of the swap you might not have done yet. |
| 5 | **Integration fit** | Does the component fit the rest of the stack (SDK language coverage, existing observability stack, existing CI, existing storage) | A best-in-class tool that doesn't fit the org's stack loses to a good-enough tool that does. |
| 6 | **Operational maturity** | SOC 2 / ISO 27001, uptime SLO, support responsiveness, incident history, roadmap velocity, community health for OSS options | A vendor that misses SLA in the middle of a release-day is a vendor whose "great features" don't matter. |

The matrix does not roll these axes into a single score. The six are qualitative when the answer is not a number; the exercise (§ chapter's exercise-03 target) has the reader author narrative answers per component per axis, so the trade-offs stay visible. The score-collapsing anti-pattern (mod-111 chapter 01) applies here too.

### Build, buy, hybrid — the three answers per component

For each component, the decision is one of three:

- **Buy.** Adopt an external product (OSS or vendor-hosted). Everything about that component's shape is inherited from the product. Trade-off: fastest to ship; largest lock-in; least extensible.
- **Build.** Own the component internally. Everything about it is authored in-house. Trade-off: slowest to ship; smallest lock-in; most extensible.
- **Hybrid.** Adopt a product for the substrate; own a shim that adds internal shape (custom rubrics, custom attribution, custom schema extensions). Trade-off: middle-ground on all axes; requires clear ownership of the shim.

Most mature eval programs run **hybrid on most components**. Trace collection buys the OTel SDK; the shim owns the internal attribute schema. Rubric runner buys the DeepEval / Promptfoo runner; the shim owns the custom rubric definitions and the versioning. Storage buys ClickHouse or a hosted Phoenix / Langfuse; the shim owns the lineage-key schema (mod-110 chapter 02).

Two component classes tend to be at the extremes:

- **Cloud-native provider eval** (component 10) is almost always **buy** — Vertex AI Evaluation, Bedrock Evaluations, and Azure AI Foundry Evaluations are integrated deeply into their respective clouds and rarely justify building.
- **Custom rubric definitions and cohort tags** are almost always **build** — they are the org's institutional knowledge and belong internally.

The matrix picks the middle for most components deliberately. A team that goes all-buy loses extensibility; a team that goes all-build burns years re-implementing what OSS gives away.

### The matrix as a document

The matrix is a document. One row per component; one column per axis; the current decision named. Concretely, a Markdown table (or a YAML the doc renders from) in the eval-program repo, versioned and signed.

Example row (excerpt):

```markdown
## Component 2: Trace storage & query

Current decision: HYBRID
  - Buy: Langfuse Cloud (EU region) for storage substrate
  - Build: internal shim `eval_platform/storage/` for lineage-key schema (mod-110 ch. 02)
  - Next refresh: 2027-Q2 (annual)

Axes:
  cost:
    list_price:      $2,400 / month (100M traces / month tier)
    integration:     6 eng-weeks (one-time, done 2026-Q1)
    operating:       ~0.2 FTE ongoing
    swap_out:        estimated 8 eng-weeks (schema-portable to Phoenix / self-hosted OTel)
  data_residency:
    region:          EU-central (Frankfurt)
    contract:        Langfuse DPA v3.2; data not used for training
    regulated_ok:    yes for EU-hosted surfaces; escalate to governance peer for US-hosted regulated
  extensibility:
    custom_attrs:    supported natively (OTel-GenAI attributes flow through)
    custom_schema:   shim required; documented in `eval_platform/storage/README.md`
  lock_in:
    schema_portable: yes (OTel-based; export supported)
    exporter:        Langfuse export API; internal snapshot script tested quarterly
  integration_fit:
    sdk_coverage:    Python + TS + Go (all app stacks covered)
    observability:   Grafana dashboards for platform slice metrics; integrates with existing on-call
  operational_maturity:
    certs:           SOC 2 Type II
    slo:             99.9% published; internal measurement matches
    support:         Slack channel + email; response < 4h business hours

Alternatives considered (2026-Q1 decision):
  - Phoenix (self-hosted): rejected — operating cost > vendor at current volume;
    revisit at 500M+ traces / month
  - Braintrust: rejected — pricing model favours smaller volumes; data residency
    less flexible for EU
  - Self-hosted OTel + ClickHouse: rejected — 12+ eng-weeks integration cost;
    revisit if org standardises on ClickHouse

Rollback / swap-out trigger:
  - Langfuse pricing shift > 30% at renewal → swap-out evaluation triggered
  - Langfuse residency change affecting EU tier → immediate escalation
  - Sustained SLO miss (< 99.5% over rolling 90d) → swap-out evaluation triggered
```

Every component gets a row like this. The document lives in `programs/platform/matrix.md`; the CI checks that every component has a row with a decision, a next-refresh date, and a rollback trigger.

### The build-vs-buy diagnostic questions

For any single component, three diagnostics help pick between build / buy / hybrid.

**Diagnostic 1 — Is this a durable differentiator?** Custom rubrics that encode the org's product commitments, cohort tags that reflect the org's user segmentation, adversary personas that reflect the org's threat model — these are durable differentiators; other orgs' versions are not substitutes. Build. Trace collectors that emit OTel-GenAI attributes are not differentiators; the OTel spec is the shape everyone converges on. Buy.

**Diagnostic 2 — Is the swap-out cost linear or step-function?** If swapping requires re-authoring six months of eval-set versions, re-scoring every historical release, or losing the entire trace history, the swap is a step-function. Step-function swaps are what cause the 18-month rewrite pain. Buy carefully; prefer components with portable schemas and export APIs; the matrix's `lock_in` axis measures this. If the swap is linear (change the SDK call, re-configure the pipeline), buy freely.

**Diagnostic 3 — Is the operating cost dominated by vendor pricing or team time?** If the vendor is $2k/month and the team spends 0.2 FTE on it, vendor pricing dominates and the buy cost is a line item. If the vendor is $0/month (OSS) and the team spends 3 FTE on it, team time dominates; the "free" OSS is more expensive than most hosted alternatives. The matrix's `cost` axis includes both.

### The vendor short-list per component

For each component's `buy` or `hybrid` row, the matrix names a **short-list** of alternatives — the two or three competitors seriously considered when the decision was made, with the reasons they were not picked. This does two things:

- **The short-list is refreshed at every review.** If a competitor's product has changed materially since the last decision (added SOC 2, added an EU region, changed its pricing model, launched a competitive feature), the short-list is refreshed. The refresh may leave the decision unchanged; that is fine — the refresh is *evidence* that the decision is defensible today, not just historically.
- **The short-list makes the swap-out cheaper.** When a trigger fires (pricing shift, residency change, sustained SLO miss), the team already has the short-list; the swap-out doesn't start from a blank sheet.

The exercise (exercise-03) has the reader author a short-list per component with a documented "not-picked because" line per alternative.

### The cadence — annual full refresh, quarterly status check

Chapter 01 named the program's rhythm; the matrix has its own beat.

- **Annual full refresh.** Every component's decision is re-evaluated from scratch. Alternatives short-listed. Axes re-scored. Decisions confirmed or changed. The refresh produces a versioned diff and a summary of what changed.
- **Quarterly status check.** A shorter review — is any component's rollback trigger being approached? Any pricing renewal in the next quarter? Any residency change announced by a vendor? Any new competitor worth adding to the next annual short-list? The quarterly check is monitoring, not re-decision.
- **Trigger-driven review.** If any rollback trigger fires (pricing shift > 30 %, sustained SLO miss, residency change), the review for *that* component happens immediately, out of cadence.

The cadence discipline is what stops the informal "should we still be on X?" from re-opening at every product review — the answer is "the matrix says the decision is valid until the next refresh in <date>; here is the trigger set." Anyone can trigger the review; nobody can re-open the decision by opinion.

### Data-residency depth — the axis worth walking

The `data_residency` axis deserves more depth than a single row because it varies by regulation and by data class, and it drives more platform decisions than most teams appreciate.

Two dimensions per component:

- **Region.** Where does the data physically sit? For a European deployment under GDPR and EU AI Act, an EU-region option is often mandatory. For a US-only deployment, a US-only option may be preferred to avoid cross-border transfer complexity. For a multi-region deployment, the vendor's multi-region shape matters.
- **Sovereignty and data-use contract.** Is data used to train the vendor's models? Is it visible to vendor personnel? What are the retention defaults? Is DPA (Data Processing Addendum) shape acceptable to the org's legal team?

Some vendors have region-locked SKUs (Langfuse EU, Braintrust regional deployments); some have universal cloud regions but data-use terms that vary by contract; some do not offer regional isolation on cheaper tiers.

The three cloud-native offerings (Vertex AI Evaluation, Bedrock Evaluations, Azure AI Foundry Evaluations) inherit their respective clouds' residency guarantees, which is usually the deepest of any option — but with the cloud-lock-in trade-off.

For an org whose customer surfaces are multi-jurisdictional, the residency axis is often what decomposes the platform. A single-vendor "eval platform" may not fit; a per-region deployment of the same platform (or per-region different platforms) may be the honest answer, and the matrix names it.

### Extensibility depth — the second axis worth walking

Extensibility looks like a boolean at first glance (can you add a custom rubric? Yes / no) and turns out to be a spectrum. The axis worth naming:

- **Native extension.** The tool provides a documented extension surface (a plugin API, a custom-evaluator abstract class, a webhook). Extensions are supported and survive upgrades.
- **Configuration extension.** The tool is configurable enough that most extensions can be expressed without writing code (a YAML rubric definition; a Colang policy in NeMo Guardrails).
- **Fork-and-modify.** The tool is OSS; extensions require forking and maintaining a delta. Works, but the maintenance tax is real.
- **Shim extension.** The tool is used as a substrate; the extension lives in an internal shim. Works well when the tool's interface is stable; brittle when it isn't.
- **No extension.** The tool doesn't extend. The org either lives without the extension or picks a different tool.

For each component the matrix picks a level of extensibility the team can sustain. A team of two cannot sustain three fork-and-modify components; it can sustain one shim-extended component well.

### The build-vs-buy anti-patterns

**"Sunk-cost gravity."** The current platform was set up two years ago; nobody is willing to swap because of the effort already spent. The matrix's job is to make the decision defensible on *future* cost, not on past cost.

**"Vendor-of-the-week."** A new tool launches; a Slack thread argues for swapping; a swap happens; six months later a newer tool launches. The matrix's cadence is the anti-body — the annual refresh is when swaps are considered, unless a trigger fires.

**"Build everything so we're not locked in."** The team owns 10 shims; the shims are undermaintained; the platform has more bugs than any vendor equivalent. Fix: build only durable differentiators; buy substrates.

**"Buy everything so we don't waste time."** The team's rubrics live in the vendor's UI; the vendor sunsets the feature; the rubrics disappear. Fix: durable differentiators are always built (even if the substrate is bought).

**"One-vendor lock-in."** The team standardises on one vendor for storage + rubric + online loop + human review; when the vendor's pricing shifts 3x, the migration is a year. Fix: component-level decisions; different vendors for different components is a healthy state.

**"Matrix set once, never refreshed."** The matrix was authored two years ago; the decisions are all "valid" because nobody has re-checked. Fix: annual refresh cadence baked into the calendar; the quarterly check catches trigger events between refreshes.

### The escalation shape when a component's decision is contested

The matrix is a document; documents are contested. Two shapes for contested decisions:

- **Ad-hoc contest.** Someone (product, eng, another team) argues in a Slack thread that a specific component's decision is wrong. The matrix owner responds by pointing at the matrix row (the axes, the alternatives-considered, the trigger set). If the contest is a new argument not covered in the matrix, it goes into the next quarterly-review agenda; if it is a repeat of an argument already covered, the matrix row is the answer.
- **Trigger-driven contest.** A trigger fires (pricing shift, residency change, SLO miss). The matrix owner starts the out-of-cadence review immediately; the review is timeboxed (usually two weeks). The team involved in the contested component participates.

Neither shape is "keep arguing until someone gives up." Both route to a decision on a documented cadence.

### A minimal walkthrough

A young org (six months into standing up the eval program) authors its first matrix. Three engineers on eval; four app surface families in scope.

**Decomposition.** They walk the ten components. Some do not apply yet (component 10, cloud-native provider eval, is out of scope because the org is multi-cloud). Nine components on the matrix.

**Trace collection and storage** get combined for now (the org's volume is small enough that Langfuse hosts both natively). Decision: **buy Langfuse Cloud EU**; short-list Phoenix (rejected for operating cost), Braintrust (rejected for residency). Next refresh: 12 months.

**Eval-set / test-case management.** Small enough today to sit in Braintrust datasets. Decision: **buy Braintrust**. Short-list: Langfuse datasets (rejected for less-mature reviewer UI at time of decision), internal Postgres (rejected for reviewer-UI absence).

**Rubric / judge runner.** The custom rubrics need code-level extensibility. Decision: **hybrid** — DeepEval as substrate, internal shim `eval_platform/rubrics/` for org-specific rubrics. Short-list: Promptfoo (rejected for lower Python API maturity at time of decision), Braintrust `Eval()` (rejected for lock-in).

**Offline gate CI integration.** GH Actions natively supported. Decision: **build** the shim over Promptfoo (already used for the rubric shim). Short-list: Braintrust CI, DeepEval CI (both rejected for wanting to keep CI internal to the rubric shim's toolchain).

**Online loop.** Reads from Langfuse's online-mode; internal alerting layer over Grafana. Decision: **hybrid** — Langfuse for scoring, internal for alert routing. Short-list: Phoenix (rejected because storage already on Langfuse), Datadog LLM Observability (rejected for cost).

**Safety infrastructure.** Llama Guard for open-source substrate, Presidio for PII detection, internal orchestration. Decision: **hybrid** — buy the classifiers, build the orchestration. Short-list: NeMo Guardrails (rejected for complexity vs. current need), OpenAI Moderation (rejected for hosted-only).

**Human review.** Argilla for annotation UI. Decision: **buy Argilla**; short-list Label Studio (rejected for reviewer-quality features), internal (rejected for team size).

**Cost & pricing accounting.** Internal shim over vendor consoles. Decision: **build**; short-list Helicone (rejected for wanting attribution keys as internal shape).

The matrix is 9 rows, one signed doc, committed to `programs/platform/matrix.md`. The annual refresh is scheduled for 12 months out; the quarterly check for 3 months out; the Langfuse renewal is 9 months out (a specific quarterly-check item).

Two years later, a trigger fires — Langfuse announces a pricing model change that would double the org's bill at current volume. The matrix owner starts the out-of-cadence review; the short-list already has Phoenix and Braintrust; the review confirms Phoenix is now competitive at the org's grown volume and swaps to self-hosted Phoenix. The swap is documented as a matrix-diff; the six-month integration effort is planned against; the eval program continues to ship gates through the migration because the migration is *planned*, not reactive.

### The end-of-chapter posture

By the end of this chapter you can:

- **Decompose the eval-platform stack** into named components — trace collection, storage, eval-set management, rubric runner, offline gate, online loop, safety infra, human review, cost accounting, cloud-native provider eval.
- Score each component on the **six axes** — cost of ownership, data residency, extensibility, vendor lock-in, integration fit, operational maturity — with narrative (not single-score) answers.
- Decide **build / buy / hybrid** per component with a **vendor short-list** and **not-picked-because** lines.
- Publish the **matrix as a document** with **rollback triggers** per component and a **refresh cadence** (annual full; quarterly check; trigger-driven review).
- Refuse re-litigation by pointing at the matrix; enqueue new arguments for the next quarterly review; run out-of-cadence reviews only when a trigger fires.

Chapter 05 opens the incident-driven investment loop — the process that turns production incidents into permanent regression fixtures the platform stack the chapter just picked will host.

## Summary

- **Decompose the eval platform into components**, not one big buy. Ten components typical: trace collection, storage & query, eval-set / test-case management, rubric / judge runner, offline gate CI integration, online loop, safety infrastructure, human review, cost accounting, cloud-native provider eval.
- Score each component on **six axes**: cost of ownership, data residency & sovereignty, extensibility, vendor lock-in, integration fit, operational maturity. No single-number roll-up.
- Three answers per component: **build**, **buy**, **hybrid**. Most mature programs run hybrid on most components — buy the substrate, build the durable-differentiator shim.
- Three **build-vs-buy diagnostics**: is this a durable differentiator? Is the swap-out cost linear or step-function? Is the operating cost dominated by vendor pricing or team time?
- The matrix is a **document** — one row per component; the decision, the short-list, the alternatives-considered lines, the rollback triggers per component.
- **Cadence**: annual full refresh (every component re-evaluated from scratch); quarterly status check (triggers monitored); trigger-driven out-of-cadence review (pricing shift, residency change, SLO miss).
- **Data residency depth** — region + sovereignty; a single-vendor "eval platform" may not fit a multi-jurisdictional org. Cloud-native provider eval (Vertex, Bedrock, Foundry) inherits its cloud's residency at the price of cloud lock-in.
- **Extensibility spectrum**: native extension, configuration extension, fork-and-modify, shim extension, no extension. Pick the level the team can sustain per component.
- **Anti-patterns**: sunk-cost gravity, vendor-of-the-week, build-everything, buy-everything, one-vendor lock-in, matrix-set-once-never-refreshed.
- **Contested decisions** route to the quarterly review (ad-hoc) or the out-of-cadence review (trigger-driven); the matrix row is the answer to repeat arguments.

Chapter 05 opens the incident-driven investment loop — the process that turns every real production incident into a permanent regression fixture, an alert, and a runbook diff, hosted on the platform stack this chapter selected.
