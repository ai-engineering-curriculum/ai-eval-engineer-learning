# Product-Side Model / System Card Slice: The Composed Artefact the Governance Peer Consumes

## Motivation

Model cards and system cards are the industry-standard way to describe an AI system to reviewers — internal, third-party, or regulatory. Mitchell et al.'s 2019 paper "Model Cards for Model Reporting" defined the model-card shape; the intervening years have added system-card variants that describe the *deployed system* rather than the raw model. Today, most orgs shipping AI at scale maintain an internal card per system (or per surface family), and the card is the reference the governance peer hands to the auditor, the customer procurement team, or the regulator when asked.

The card is *not* the eval program's artefact. It is the **governance peer's artefact**, folded into the org-wide governance surface, signed by the peer's owner, and formatted per the org's chosen template (often a superset of the Mitchell shape adapted for the org's regulatory posture). The `ai-evaluation-engineer` (Governance, L25) peer and `ai-governance-analyst` own the card end-to-end.

What the eval program *does* own is the **card slice** — the eval-evidence contribution to the card, per app surface family, in a shape the governance peer folds in without rewrites. Chapter 03's delegation contract with the governance peers named the card slice as a first-class artefact; this chapter walks its shape.

Two symptoms show the card slice is missing:

- **The governance peer authors the eval sections from scratch.** They read the eval program's dashboards, take notes, paraphrase, and land the paraphrase in the card. The paraphrase drifts from the real numbers; the peer's authorship burns time; the eval program has no visibility into what the card says.
- **The card is stale within weeks of publication.** The numbers were pulled at authoring time; they are frozen; the release-gate outcomes, incident-ledger progress, and framework mappings from the next month are not in the card. The card becomes a historical artefact rather than a live description.

Both symptoms are integration failures. This chapter closes them.

## Core concepts

### The card slice at a glance

A card slice is the eval-program contribution to the org-wide card for one app surface family. It is not the whole card; the whole card includes governance, policy, data provenance, human-oversight, and vendor / third-party sections the governance peer authors. The slice is the eval-evidence portion.

Typical slice sections (per family):

- **Intended use.** What the surface does; who the users are; the deployment context. Mostly product-authored; the eval program adds the evaluation-relevant framing.
- **Evaluation results — per surface, per rubric, per cohort.** The current release's rubric scores against the pre-registered thresholds, per cohort, with the mod-111 trade-off report's evidence as the underlying artefact.
- **Model capability profile.** The model-altitude capability profile from the `model-evaluation-engineer` peer (chapter 03's delegation contract), summarised for the card reader. Sourced by reference, not re-derived.
- **Safety evaluation results.** The mod-108 safety report's headline results — OWASP LLM Top-10 posture, jailbreak-resistance rate, PII-leak rate, guardrail effectiveness. Sourced from the safety report.
- **Known limitations and mitigations.** From the mod-111 report + the mod-108 findings + the investment ledger (chapter 05). Every known failure mode with its mitigation.
- **Monitoring and change process.** From chapter 02 — the release-gate architecture summary; from chapter 05 — the investment ledger's process shape.
- **Recent incident summary.** From the investment ledger — the P0 / P1 incidents on this family in the last N months, with severity, root-cause class, and mitigation status.
- **Reproducibility artefacts.** Pointers to the evidence — the release-gate history log, the incident ledger, the mod-110 platform-slice queries the reader can run to reproduce.

The slice is one document per family (Markdown, per the eval-program repo convention); the governance peer folds it into the org-wide card by reference or by copy — the eval program does not care which as long as the reference is stable.

### The slice-generator pattern — never author the slice by hand

The single most important discipline of the card slice is that it is **generated**, not authored. The eval program's monthly ceremony produces a fresh slice per family from the underlying artefacts. The generator reads:

- **The current release-gate history log.** For every surface in the family, the last N releases' gate outcomes.
- **The current mod-111 trade-off report** per surface (per shipped candidate).
- **The current mod-108 safety report** per surface.
- **The current investment ledger** for incidents affecting the family.
- **The current framework mapping** (chapter 07) for the family.
- **The current model-capability-profile snapshot** from the model-eval-eng peer (chapter 03's contract).

The generator writes the slice document to a well-known path (e.g., `programs/card_slices/customer_facing_support_agents/2026-Q3.md`); the governance peer's card-assembly pipeline reads the well-known path.

Two properties of the generator:

- **Deterministic.** Same inputs → same slice. The generator does not paraphrase; it composes.
- **Timestamped.** Every slice carries the timestamp of its inputs (which release-gate log, which safety report version, which trade-off report version). The card reader can trace every number back to the source artefact.

Hand-authored slices drift from the underlying evidence; generated slices do not. The exercise for this chapter has the reader author the generator, not the slice itself.

### The Mitchell shape and its evolution

The Mitchell et al. 2019 model-card shape has nine sections: model details, intended use, factors, metrics, evaluation data, training data, quantitative analyses, ethical considerations, caveats and recommendations. Most current card templates in industry are extensions of this shape.

**Provider-published cards worth studying** as reference shapes (as of the 2026 landscape — verify current):

- **Anthropic Claude model cards / system cards.** Extended Mitchell shape with sections on constitutional AI, deployment, RSP tier.
- **OpenAI system cards** (GPT-4, GPT-4o, o1, and successor releases). Extended shape with sections on capability evaluation, safety evaluation, adversarial red-teaming, preparedness-framework tier.
- **Google DeepMind Gemini model cards / technical reports.** Extended shape with sections on capability evaluation, safety evaluation, responsible-development choices.
- **Meta Llama model cards.** Community-oriented shape with sections on training, use policy, evaluation.

<!-- needs-research: on the next research cycle, re-verify the provider-published model card / system card sources — the shapes and section names evolve per release; sync the reference list above to current published cards. -->

The important thing is not any specific template. The important thing is that the org's chosen template has a shape, the governance peer publishes it, and the eval program's slice fits into it. The chapter's exercise-05 target has the reader map the slice's sections to whatever the org's template looks like.

### Section-by-section walk

The following is a section-by-section walk of a typical slice for one family. Every section names its evidence source and the artefact the slice pulls from.

**§1 Intended use (family-level).** Two – three paragraphs. The surfaces in the family; the users; the deployment context; what the surfaces are and are not designed for.

Source: family-level product doc + release-gate policy (chapter 02). Eval program adds the "the following surfaces are gated by policy X" framing.

**§2 Evaluation results per surface.** For every surface in the family:

- The pre-registered decision rule from the last release (mod-111 chapter 02's shape).
- The gate outcomes at offline / canary / ramp / post-ship for the last release.
- The rubric scores per cohort, with the pre-registered threshold and the outcome.

Source: chapter 02's release-gate history log; mod-111's trade-off report per shipped candidate.

**§3 Model capability profile.** For every model snapshot in production on the family:

- Model-altitude benchmarks (MMLU, GPQA, HumanEval, MMMU, LiveCodeBench, or whichever suite the model-eval-eng peer publishes).
- Vendor-published or peer-measured capability numbers.
- Any dangerous-capability signal from the peer.

Source: chapter 03's delegation contract with `model-evaluation-engineer` — the capability profile snapshot. **By reference; do not re-run.**

**§4 Safety evaluation results.** For the family:

- OWASP LLM Top-10 posture per surface (from mod-108 chapter 05).
- Jailbreak-resistance rate on the risk-eng peer's red-team corpus (from mod-108).
- PII-leak rate on the mod-108 privacy suite.
- Guardrail effectiveness (from mod-108's guardrail evaluations).
- Attack-signal enrichment from the `ai-infra-security` peer (from chapter 03's contract).

Source: mod-108 safety report per surface; chapter 03's contract with `ai-risk-engineer` and `ai-infra-security`.

**§5 Known limitations and mitigations.** For the family, a list of known failure modes:

- Every root-cause class from the investment ledger (chapter 05) currently affecting the family.
- Per class: the failure shape, the current mitigation (guardrail, prompt-side, retrieval-side, rate-limit, etc.), the residual risk, the monitoring alert that catches recurrence.

Source: investment ledger (chapter 05); mod-108 findings; the delegation-contract follow-ups.

**§6 Monitoring and change process.** For the family:

- The release-gate architecture summary (chapter 02).
- The four gates, thresholds' provenance, rollback criteria.
- The change-class → gate-map summary.
- The quarterly refresh cadence.
- The investment ledger's monthly review cadence (chapter 05).

Source: chapter 02's policy document; chapter 05's ledger process.

**§7 Recent incident summary.** For the family, incidents in the last N months (N usually 3 – 12 depending on card audience):

- Incident id, severity, date, surface, root-cause class, mitigation status.
- For each incident closed with a permanent fixture: the fixture reference, the alert reference, the runbook-diff reference.

Source: investment ledger (chapter 05).

**§8 Reproducibility artefacts.** For the family:

- The eval-set version(s) currently gating the surface (mod-106 replay bundle references).
- The mod-110 platform-slice queries a reader can run to reproduce the numbers (`SELECT ... FROM eval_results WHERE eval_run_id = ...`).
- The release-gate history log's location.
- The investment ledger's location.
- The card slice's own timestamp and hash.

Source: mod-110 platform slice; chapter 02's log; chapter 05's ledger; the slice generator itself.

### Update cadence — monthly diff, quarterly ceremony

The slice is regenerated **monthly** by the generator; the governance peer folds the update into the card on their own cadence (often monthly, sometimes quarterly depending on the audience).

Two triggers can push a slice update out-of-cadence:

- **A P0 or P1 incident closes.** The investment-ledger row for the incident is authored; the slice's §5 (limitations) and §7 (incidents) sections gain a row; the slice is regenerated and the governance peer is notified.
- **A gate policy refresh (chapter 02's quarterly refresh).** The thresholds change; the slice's §6 (monitoring and change process) needs to reflect; the slice is regenerated.

The quarterly **card slice diff review** (chapter 01's monthly rhythm elevated to quarterly for the card audience) is a joint session between the eval program owner and the governance peer. Agenda:

- **What changed on the slice this quarter?** Section by section.
- **Are the sections still fit-for-purpose?** Any card audience (internal, third-party, regulatory) that needs a different framing?
- **Is the framework mapping (chapter 07) reflected accurately?** Any framework updates since the last review?
- **Any drift between the slice and the governance peer's card?** The peer should be folding the slice by reference; if the peer's card has language that is not in the slice, the drift is a signal — either the slice needs a section the peer is authoring, or the peer's authoring is duplicating work.

The review produces a diff and (if needed) a slice-generator update.

### The "handed cleanly to the governance peer" contract

Chapter 03's delegation contract with the governance peers named the slice as a first-class artefact. The contract-side requirements the slice honours:

- **Well-known path.** The generator writes to a path the governance peer's card-assembly pipeline reads. If the path changes, the contract is updated; ad-hoc "here's the slice, sorry, forgot to email it" is not the shape.
- **Schema fit.** The slice sections map 1:1 to sections in the governance peer's org-wide card template. If the template changes, the generator is updated in the same PR that the template is updated.
- **Deterministic + timestamped.** Peer can rerun the generator against the same inputs and get the same output. Every number carries its source hash.
- **Zero rewrites.** The peer folds the slice by reference (or by copy without changes). If the peer is rewriting sections, the slice is not fit for consumption; fix the generator.
- **Change notification.** Slice updates that materially change a section (a new incident row, a threshold change, a new limitation) trigger a notification to the peer.

The exercise for this chapter has the reader author the generator and confirm the peer-consumption path is intact.

### The reader-audience framing

Card slices land in cards; cards land in front of readers. Different readers need different framings; the slice is authored for one canonical framing (the governance peer's template) but *should* be re-framed by the peer for different audiences.

Reader audiences (typical):

- **Internal risk review.** Detailed evaluation numbers; per-cohort breakdowns; incident ledger detail. The reader is a risk officer who wants the substrate.
- **Third-party procurement / customer security review.** Aggregated numbers; posture summaries; certifications; roadmap for known limitations. The reader is a customer's security team wanting reassurance without the substrate.
- **Regulatory submission.** Framework-mapped evidence; specific citations to NIST AI RMF actions, EU AI Act articles, ISO/IEC 42001 controls (chapter 07). The reader is a regulator or an auditor.
- **Public model card.** A curated subset — the intended use, the safety evaluation results, the known limitations — without the confidential evaluation set details or the incident-specific detail. The reader is anyone.

The slice is authored to be **subsettable** — each section has an audience-tag (`internal`, `third_party`, `regulatory`, `public`) so the governance peer can compose per-audience cards without re-authoring.

### Anti-patterns to avoid

**"Hand-authored slice."** The slice is written each cycle from scratch by reading dashboards. Drift from evidence is guaranteed; the peer's fold-in is high-friction. Fix: generator.

**"Slice is the peer's."** The eval program hands the peer the raw artefacts and the peer authors the slice. The peer's authorship is high-effort; the peer's paraphrase drifts. Fix: the slice is *always* the eval program's; the *card* is the peer's.

**"Slice is a dashboard link."** The slice is a link to a live dashboard; the card reader clicks; the dashboard shows numbers from a different point in time. Fix: the slice is a *timestamped document*, not a live view.

**"Slice covers the whole card."** The slice grows to include governance, policy, and vendor sections that are the peer's remit. The peer is over-loaded; the slice's shape drifts. Fix: the slice is the eval-evidence portion only.

**"Slice audiences confused."** A single slice is expected to serve internal risk review and public model card. The public card leaks internal detail; the internal review lacks depth. Fix: audience-tagged sections; peer composes per-audience.

**"Slice not diffed."** The slice is regenerated; the peer folds it; nobody reviews what changed. A stale limitation persists; a resolved limitation is still called out. Fix: quarterly slice-diff review.

### A minimal walkthrough

The eval program owner for a customer-facing support-agent family stands up the card slice.

**Generator authoring.** The eval-program repo grows a `programs/card_slices/customer_facing_support_agents/generate.py` script. It reads:
- `programs/gates/customer_facing_support_agents/history/*.json` — gate outcomes.
- `mod-111 trade-off reports` per surface — latest per-shipped-candidate.
- `mod-108 safety reports` per surface — latest.
- `programs/investment_ledger.yaml` — filtered to the family.
- `programs/framework_mapping/customer_facing_support_agents.yaml` — for §7 mapping (chapter 07).
- The model-eval-eng peer's capability profile snapshot from mod-110's `capability_profiles` table.

**First run.** The generator produces `programs/card_slices/customer_facing_support_agents/2026-Q3.md`. Eight sections. Every number sourced. Timestamp footer. Governance peer folds it into the org-wide card for Q3.

**Monthly ceremony.** The generator runs on the first of each month; the slice is refreshed; a diff is posted to the eval-program channel with the changed sections highlighted. The peer picks up the change in their next fold cycle.

**A P0 fires mid-quarter.** The investment-ledger row is opened; the retrospective closes; the ledger row transitions to *landed*. The next monthly regeneration adds a row to §5 (limitations) with the mitigation and to §7 (incidents). The governance peer is notified out-of-cadence; the card's next update covers the change.

**Quarterly review.** The eval program owner and the governance peer walk the slice section by section. Two changes emerge — the peer's template gained a new section on human-oversight; the eval program's slice needs an §8b for the human-review workflow from mod-109. The generator is updated in the same PR the template updates.

Six months in, the card slice is a load-bearing artefact — the peer's card-assembly pipeline reads it directly, the internal review has stopped auditing the eval-program dashboards separately, and the incident-to-card latency is measured in weeks rather than months.

### The end-of-chapter posture

By the end of this chapter you can:

- Author a **card-slice generator** for one app surface family that composes the eval evidence into an eight-section Markdown slice with source-hash provenance on every number.
- Wire the generator to the **monthly regeneration cadence** and the **out-of-cadence triggers** (P0 / P1 close, gate policy refresh).
- Land the slice at the **well-known path** the governance peer's card-assembly pipeline reads.
- Author **audience-tagged sections** so the governance peer can compose per-audience cards without rewrites.
- Run the **quarterly slice-diff review** with the governance peer — walk the diff, catch drift, update the generator.

Chapter 07 opens the framework mapping — the layer above the card slice that ties every program artefact to NIST AI RMF, EU AI Act, and ISO/IEC 42001 controls.

## Summary

- A **card slice** is the eval-program contribution to the org-wide model / system card for one app surface family. The whole card is the **governance peer's** artefact; the slice is the eval-evidence portion.
- The slice is always **generated**, never hand-authored. A monthly generator reads the release-gate history, the trade-off reports, the safety reports, the investment ledger, the capability profile, and the framework mapping, and produces a timestamped Markdown slice.
- Eight sections typical: intended use, evaluation results per surface, model capability profile, safety evaluation results, known limitations and mitigations, monitoring and change process, recent incident summary, reproducibility artefacts.
- The **Mitchell 2019 model-card shape** is the industry substrate; provider-published cards (Anthropic, OpenAI, Google DeepMind, Meta) are the current reference points; the org's chosen template extends the shape.
- **Update cadence**: monthly regeneration (default); out-of-cadence for P0/P1 incident close or gate policy refresh; quarterly diff-review with the governance peer.
- **Contract-side requirements**: well-known path, schema fit to the peer's template, deterministic + timestamped output, zero rewrites by the peer, change notification.
- **Audience-tagged sections** let the peer compose per-audience cards (internal risk review, third-party procurement, regulatory submission, public model card) without re-authoring.
- **Anti-patterns**: hand-authored slice, slice authored by the peer, slice-as-dashboard-link, slice covering the whole card, audience-confused sections, slice not diffed.

Chapter 07 opens the framework mapping — the layer that ties the program's artefacts to NIST AI RMF, EU AI Act, and ISO/IEC 42001 controls.
