# exercise-03: Build Vs Buy Eval Platform Matrix

**Estimated effort:** 3 hours

## Objective

Author the chapter 04 **build-vs-buy platform matrix** for your org's eval stack — decompose the stack into the ten components the chapter names, score each on the six evaluation axes with narrative answers, pick a build / buy / hybrid decision per component with a vendor short-list, pin rollback triggers per component, and schedule the annual full-refresh plus the quarterly status check. The matrix survives one component's vendor going out of business and the "should we still be on X?" question being asked at every product review.

By the end of the exercise the eval program's platform decisions are a signed document — `programs/platform/matrix.md` plus `programs/platform/components/*.yaml` — the eval program owner can defend against last cycle's evidence. The platform-churn failure shape from chapter 01 (rewrite every 18 months because the last decision was made informally) becomes blocked by a cadence; swaps happen when a rollback trigger fires or when the annual refresh says they should, not when a Slack thread wins.

## Prerequisites

- Chapters 01 and 04 of this module.
- Exercise 01 (so the gate policy's references to CI, canary, and online loop machinery map onto specific matrix components).
- A working inventory of your current eval-stack tooling — which trace backend you use today, which rubric runner, which eval-set manager, which human-review tool. If the inventory is in someone's head, the first set-up step surfaces it.
- One finance partner (or an informal cost owner) who can share current invoices and renewal dates for the vendor components. "Cost of ownership" requires real numbers.
- Access to the vendor pricing pages for every vendor on your short-list — the matrix cites current pricing with the fetch timestamp.
- A legal / data-residency contact (even informally) if any surface is in regulated scope — the `data_residency` axis requires their input.
- Python 3.11+; `pyyaml` and a Markdown table renderer (`mdformat`, `tabulate`, or a hand-written Jinja template).

## Set-up

1. Create the platform-matrix directory:

   ```
   programs/platform/
   ├── matrix.md                             # the rendered matrix (one row per component)
   ├── components/
   │   ├── 01-trace-collection.yaml
   │   ├── 02-trace-storage-and-query.yaml
   │   ├── 03-eval-set-management.yaml
   │   ├── 04-rubric-judge-runner.yaml
   │   ├── 05-offline-gate-ci.yaml
   │   ├── 06-online-eval-monitoring.yaml
   │   ├── 07-safety-guardrail-infra.yaml
   │   ├── 08-human-review-annotation.yaml
   │   ├── 09-cost-pricing-accounting.yaml
   │   └── 10-cloud-native-provider-eval.yaml
   ├── shortlist/
   │   ├── vendor-fetch-notes/
   │   │   ├── langfuse-2026-10-XX.md
   │   │   ├── phoenix-2026-10-XX.md
   │   │   ├── braintrust-2026-10-XX.md
   │   │   └── <others>-2026-10-XX.md
   │   └── vendor-roster.yaml              # vendors considered across components
   ├── triggers/
   │   └── rollback-triggers.yaml          # cross-component trigger registry
   ├── schema/
   │   └── component.schema.json
   ├── validate/
   │   ├── validate_component.py
   │   ├── validate_coverage.py              # every stack need is covered by one component
   │   └── validate_cadence.py               # every component has a next_refresh date in the future
   ├── render/
   │   └── render_matrix.py                  # components/*.yaml → matrix.md
   ├── history/
   │   ├── 2026-Q4-initial.md
   │   └── reviews/
   │       └── .gitkeep
   └── README.md
   ```

2. Author `schema/component.schema.json` enforcing the per-component row shape:

   - `id` (integer, 1 – 10)
   - `component` (string, enum of the ten chapter 04 components)
   - `description` (one-paragraph)
   - `current_decision` (enum: `build`, `buy`, `hybrid`)
   - `rationale_summary` (one-paragraph summary of why this decision)
   - `axes` (object with the six axes each as a block):
     - `cost_of_ownership` (list_price, integration, operating, swap_out)
     - `data_residency` (region, sovereignty, contract, regulated_ok)
     - `extensibility` (level: `native` / `configuration` / `fork_and_modify` / `shim` / `none`; notes)
     - `vendor_lock_in` (schema_portable: bool; exporter; swap_estimate)
     - `integration_fit` (sdk_coverage; existing_stack_notes)
     - `operational_maturity` (certs; slo; support; roadmap_velocity)
   - `vendor_shortlist` (list of `{ name, status: picked | rejected, not_picked_because?, fetch_notes_ref }`); exactly one `status: picked` for `buy` / `hybrid` or `none` for `build`
   - `rollback_triggers` (list of `{ name, condition, action, review_window_days }`)
   - `next_refresh` (ISO date; annual by default)
   - `next_status_check` (ISO date; quarterly)
   - `signed` (list of `{ role, name, date }`); roles: `eval_program_owner`, `platform_owner`, `finance_partner`, `legal_or_data_residency_contact` (last two required if the component has material cost or residency implications)

3. Fill one `components/*.yaml` per chapter 04 component. The ten rows (chapter 04 § "Decompose the platform into components"):

   | # | Component | Common options to consider |
   |---|---|---|
   | 1 | Trace collection | OpenTelemetry SDKs, Phoenix collector, Langfuse SDK, Weave client, Braintrust logs, provider SDK auto-instrumentation |
   | 2 | Trace storage & query | Phoenix (Arize), Langfuse, Braintrust, Weave, Humanloop, Galileo, Patronus, self-hosted OTel + ClickHouse / DuckDB |
   | 3 | Eval set / test case management | Braintrust datasets, Langfuse datasets, Humanloop datasets, Phoenix datasets, Argilla, DVC + Git, internal Postgres |
   | 4 | Rubric / judge runner | Promptfoo, DeepEval, RAGAS, Braintrust `Eval()`, Phoenix evaluators, Langfuse, internal harness |
   | 5 | Offline gate CI integration | Promptfoo GH Actions, Braintrust CI, DeepEval CI, internal GH / GitLab shim |
   | 6 | Online eval / monitoring loop | Langfuse, Phoenix, Arize, Galileo, Patronus, Datadog LLM Observability, New Relic AI Monitoring, internal |
   | 7 | Safety / guardrail infrastructure | Llama Guard, NeMo Guardrails, Guardrails AI, Presidio, OpenAI Moderation, Azure AI Content Safety, garak, PyRIT |
   | 8 | Human-review / annotation | Argilla, Braintrust human review, Humanloop, Label Studio, internal |
   | 9 | Cost & pricing accounting | Vendor consoles + internal shim, Helicone, Langfuse, Braintrust |
   | 10 | Cloud-native provider eval | Google Vertex AI Evaluation, AWS Bedrock Evaluations, Azure AI Foundry Evaluations |

   For every component on `buy` or `hybrid`: fetch the vendor's current pricing page, note the fetch timestamp, and save the notes to `shortlist/vendor-fetch-notes/<vendor>-<date>.md`. Document the pricing numbers and the DPA / residency terms you can confirm; mark anything you cannot confirm with `<!-- needs-research: ... -->` so the review loop picks it up.

4. Fill `shortlist/vendor-roster.yaml` with every vendor you considered across the matrix, de-duplicated. Each vendor has `name`, `components_evaluated_for`, `components_picked`, `components_rejected_with_reason`, `operational_notes` (SOC 2, residency SKUs, support channels, roadmap signals). The roster is the one place to look up "did we ever look at X?"

5. Fill `triggers/rollback-triggers.yaml` with every component's trigger set in one registry. Each trigger: `component`, `name`, `condition`, `action` (`out_of_cadence_review` / `escalate_to_platform_owner` / `kickoff_swap_evaluation`), `escalation_contacts`, `window_days`. The registry is what the monitoring job reads to catch triggers between quarterly checks.

6. Author `render/render_matrix.py` that loads all ten `components/*.yaml`, validates each, and emits `matrix.md` as a Markdown document with:
   - A one-page summary table (component × decision × next refresh × trigger count)
   - A per-component section with the axes narrative, the vendor short-list, the rollback triggers, and the signer line
   - A footer with the overall `next_full_refresh`, `next_quarterly_check`, and the trigger-count-by-component mini chart

7. Fill `history/2026-Q4-initial.md` with the authoring note: which components are in bootstrap state (no vendor selected because the component is not yet live), which have `<!-- needs-research: -->` markers awaiting verification, and the first quarterly-check date on the calendar.

## Requirements

Produce a PR against your eval-program repo that adds:

1. **Ten `components/*.yaml`** — one per chapter 04 component; each validates against `schema/component.schema.json`; each has the six axes populated with narrative answers (never a single roll-up score), a current decision (`build` / `buy` / `hybrid`), a vendor short-list with `not_picked_because` lines, rollback triggers, and `next_refresh` + `next_status_check` dates.
2. **`schema/component.schema.json`** — the schema; validation errors name the missing field.
3. **`validate/validate_component.py`** — schema + semantic validation:
   - For `buy` or `hybrid`, exactly one short-list entry is `status: picked`.
   - For `build`, the short-list is `[]` or every entry is `status: rejected` with a `not_picked_because`.
   - Every rejected entry has a `not_picked_because` string ≥ 20 characters.
   - Every component with material cost (list_price > $500 / month OR integration > 2 eng-weeks) has a `finance_partner` signer.
   - Every component with cross-border data flow has a `legal_or_data_residency_contact` signer.
   - `next_refresh` is in the future and ≤ 15 months out (annual + buffer).
4. **`validate/validate_coverage.py`** — cross-matrix check:
   - Every chapter 04 component appears exactly once (ids 1 – 10; no gaps, no duplicates).
   - Components that depend on the gate policy's bindings (CI, canary, online loop) resolve to the gate policy's binding doc (exercise 01's `bind/*`).
   - Every vendor in `shortlist/vendor-roster.yaml` is referenced by at least one component.
5. **`validate/validate_cadence.py`** — every component has `next_refresh` and `next_status_check` populated; every rollback trigger has an `action`.
6. **`shortlist/vendor-fetch-notes/*`** — one note per vendor on a `buy` / `hybrid` short-list, dated, with the fetch timestamp. Pricing numbers, DPA / residency terms, support contact, incident history (if any). Unverified facts tagged with `<!-- needs-research: ... -->` so the quarterly check can refresh.
7. **`shortlist/vendor-roster.yaml`** — the de-duplicated vendor roster.
8. **`triggers/rollback-triggers.yaml`** — the cross-component trigger registry; every entry maps back to a component's trigger list.
9. **`render/render_matrix.py`** — loads the components; renders `matrix.md`; stable byte-for-byte output across runs (so diffs are meaningful). The script accepts a `--check` mode that fails CI if the rendered `matrix.md` is out of sync with the YAML files.
10. **`matrix.md`** — the rendered document; signed by the eval program owner and the platform owner; readable in one sitting; every component's row answers "what did we pick, why, what would make us swap, and when do we review next?"
11. **CI integration** — the validators run on every PR touching `programs/platform/`; `render_matrix.py --check` runs on every PR to prevent drift between the YAML and the rendered matrix.
12. **`history/2026-Q4-initial.md`** — the authoring note; what is in bootstrap state; what `<!-- needs-research: -->` markers were left; the first quarterly-check calendar entry.
13. **Calendar entries** — the next full-refresh date (12 months out) and the next quarterly-check date (3 months out) on the shared calendar; owner is the eval program owner (or the explicit platform owner if your org splits the role).

## Starter guidance

- **Decompose first; score second.** The single biggest mistake is treating "the platform" as a monolith. If one of your ten `components/*.yaml` ends up with "we use Vendor X for everything," you have not decomposed. Different components have different swap costs; conflating them hides where the lock-in is.
- **Narrative answers, not single scores.** Chapter 04 is explicit: no roll-up score. "Cost: 8/10" is not a trade-off; "Cost: $2,400/mo list + 6 eng-weeks integration (one-time, done 2026-Q1) + ~0.2 FTE ongoing; swap-out estimated 8 eng-weeks; schema-portable to Phoenix" is a trade-off. The six-axis narrative is the point.
- **Document the alternatives you rejected, not just the one you picked.** The `not_picked_because` lines are what makes the matrix defensible when someone asks "why aren't we on X?" at a product review. "We didn't look at X" is a different (and worse) answer than "We looked at X and rejected it in 2026-Q4 because its residency SKUs did not cover EU-regulated surfaces."
- **Fetch pricing today; cite the timestamp.** Pricing moves quarterly across most vendors. A matrix that cites "list price $2k/mo" without a fetch date is stale within a quarter. The `shortlist/vendor-fetch-notes/<vendor>-<date>.md` pattern makes the staleness observable.
- **`<!-- needs-research: ... -->` is a feature.** When you cannot confirm a vendor's SOC 2 status, residency guarantees, or roadmap commitment, mark it and move on. The next quarterly check reads the markers and refreshes.
- **Hybrid is often correct.** Chapter 04's observation holds: most mature eval programs run hybrid on most components — buy the substrate (OTel SDKs, DeepEval, Argilla), build the shim (lineage-key schema, custom rubric registry, cohort tags). All-buy loses extensibility; all-build burns years. The matrix picks the middle deliberately.
- **Custom rubrics and cohort tags are almost always `build`.** They encode the org's institutional knowledge; vendor substitutes do not exist. The matrix defaults to `build` on component 4 and 8 unless there is a specific reason not to.
- **Cloud-native provider eval is almost always `buy` or `out of scope`.** Vertex / Bedrock / Foundry are deeply integrated with their cloud; building a competitor is wasted effort. If your org is single-cloud, the component is `buy`; if multi-cloud, the component is often `out_of_scope` (named as such in `current_decision`).
- **Rollback triggers are specific numbers.** `pricing shift > 30% at renewal` is a trigger. `cost gets too high` is not. `sustained SLO miss < 99.5% over rolling 90d` is a trigger. `vendor goes down a lot` is not. The specificity is what makes the trigger actionable.
- **Finance / legal signers only where material.** The validator requires a finance signer if `list_price > $500/mo OR integration > 2 eng-weeks` and a legal signer if the component has cross-border data flow. Avoid signer sprawl on low-stakes components; the trade-off is lower friction for small components, deliberate rigor for big ones.
- **The matrix is the answer to re-litigation.** When "should we still be on X?" comes up at a product review, the answer is "the matrix says the decision is valid until `<next_refresh>`; here's the trigger set." Anyone can trigger a review by naming a trigger; nobody can re-open the decision by opinion.
- **The quarterly check is monitoring, not re-decision.** The quarterly check reads the rollback triggers and the vendor-fetch-notes, scans for recent announcements (pricing changes, residency announcements, feature launches), and either leaves the matrix alone or kicks off an out-of-cadence review for a specific component. Full re-decision only on the annual refresh.

## Acceptance criteria

You are done when:

- Ten `components/*.yaml` files exist, one per chapter 04 component, each validating against the schema.
- Every component has the six axes populated with narrative answers; no single-number roll-up scores.
- Every `buy` / `hybrid` component has at least two vendors on the short-list (one picked, at least one rejected with a `not_picked_because` ≥ 20 characters).
- Every short-listed vendor has a fetch-note in `shortlist/vendor-fetch-notes/` with a date within the last 30 days of the authoring PR.
- `shortlist/vendor-roster.yaml` deduplicates vendors across components; every roster entry is referenced by at least one component.
- `triggers/rollback-triggers.yaml` has an entry per component trigger; every trigger has a specific condition (number or event, not a vibe).
- `render/render_matrix.py` produces `matrix.md` byte-for-byte reproducibly; `--check` passes against the committed `matrix.md`.
- `matrix.md` is signed by the eval program owner and the platform owner; components with material cost have the finance signer; components with cross-border data flow have the legal / residency signer.
- The validators and `render --check` run in CI on every PR touching `programs/platform/`; a deliberate break (remove a `not_picked_because`, set a past `next_refresh`, add a vendor to a component without a fetch note) fails the CI.
- Shared-calendar entries exist for the next quarterly check (within 90 days) and the next annual full refresh (within 365 days), each owned by the eval program owner (or platform owner if split).
- A new reader opening `matrix.md` can answer "what did we pick, why, when do we review again, and what would make us swap?" for every component without opening another file.

## Stretch goals

- **Pricing drift CI job.** A scheduled job re-fetches each vendor's pricing page weekly, diffs against the stored fetch note, and opens a tracker ticket if the price has materially changed (configurable threshold). Catches a pricing shift before the renewal month.
- **Residency map visualiser.** Parse the ten components' `data_residency` blocks and emit a per-surface map (surface → data path → region → vendor). The visualiser surfaces cross-border transfers the matrix might not have made obvious.
- **Swap-cost estimator.** For each `buy` / `hybrid` component, estimate the swap-out cost in eng-weeks under three scenarios (`schema-portable` / `partial-migration` / `full-rebuild`); maintain the estimate as a formula over current traffic volume so the number stays current as volume grows.
- **Alternatives-considered diff.** When a quarterly check or annual refresh changes a component's decision, generate a diff of the short-list showing which vendors joined, which left, and why. The diff is what the review walks through.
- **Trigger simulation.** A dry-run harness that fires each rollback trigger against the current component state and reports what action would be taken; useful for calibrating thresholds without waiting for a real event.
- **Cross-component vendor exposure.** When one vendor is picked for multiple components, flag the aggregate exposure (both lock-in and single-point-of-failure risk). The chapter 04 "one-vendor lock-in" anti-pattern is observable here.
- **Public-component reference.** For OSS components (Phoenix, Langfuse, Promptfoo, DeepEval, Argilla, Llama Guard, etc.), link the fetch note to the project's GitHub release cadence, maintainer activity, and open-issue count. The matrix's `operational_maturity` axis gets a real evidence base.

## What this exercise does *not* cover

You are not authoring the release-gate architecture (exercise 01), the delegation contracts (exercise 02), the investment ledger (exercise 04), or the card slice generator (exercise 05). You are shipping the *build-vs-buy platform matrix* — the artefact that pins the eval stack's decisions on a cadence so the chapter 01 platform-churn failure stops being how the eval team spends 30 % of its time.
