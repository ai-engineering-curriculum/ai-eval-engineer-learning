# Vendor vs. In-House Review, and the Hybrid

## Motivation

Chapters 02 – 05 define the work. This chapter defines the *labour model*: who actually sits at the workbench, who labels the thousand rows per week, who adjudicates the disagreements, who rotates the gold set. Three paths exist — fully in-house, fully vendor, or hybrid — and the pick shapes every other decision in the module (cost per label, SLA feasibility, data-residency posture, rubric evolution speed, exit cost).

Three grounded reasons the pick is a module-level concern and not a procurement-only concern.

- **The labour model is coupled to the eval mechanics.** An in-house panel can revise the rubric with the eval engineer in the same meeting; a vendor pool revises the rubric on contractually-defined cadence (a change notice, a re-training cycle, a re-measurement). The rubric iteration speed in chapter 04 and the rotation cadence in chapter 05 are both functions of the labour model. Pick without reading those chapters and the labour model constrains the eval practice silently.
- **Cost per label is not the only cost.** Vendor rate cards look cheap in isolation — until a team accounts for the management overhead, the data-residency architecture, the exit-cost reality of switching vendors, and the in-house seat-time the vendor relationship itself consumes. The cost model in mod-111 reads the full picture; chapter 06 is where this module writes it.
- **Vendor-methodology audits are peer work.** This module decides *which vendor fits the operational posture*. Whether the vendor's own labelling methodology meets the methodology bar — how they select and validate labellers, how they compute and report IAA, how they audit label noise — is a `model-evaluation-engineer` artefact. The module consumes the peer's audit; it does not re-derive it.

This chapter walks the trade-off matrix (seven axes), surveys the vendor landscape as *shape templates* rather than endorsements, sketches the in-house build-cost, describes the hybrid pattern most product teams converge to, and names the delegation points to the peer track.

## Core concepts

### The seven trade-off axes

The decision is seven-dimensional. Make the pick on all seven at once; optimising for one blows up a different one.

- **Cost per label.** Vendors publish rate cards or quote per-label pricing; in-house labour is at seat cost (salary + benefits + overhead, divided by labelled rows per unit time). Vendors look cheaper per row at steady state; in-house looks cheaper once the pool is amortised and the rows are not routinely idle.
- **Latency to first label.** Time between a row entering the queue and the first labelled verdict. In-house pools have faster latency on urgent queues (the reviewer is on Slack); vendor pools have better throughput on steady-state queues but slower first-label latency when the queue is bursty.
- **Data-residency and PII posture.** Does the row contain PII, sensitive content, or protected categories? Can the row leave the production region? Vendor labellers in other jurisdictions are an issue; some vendors offer regional labelling pools (EU-only, US-only) at a premium; some content (e.g. medical records, certain financial data) cannot leave the enterprise boundary under any vendor contract.
- **Vertical / language coverage.** Does the rubric require domain expertise (medical, legal, financial, security-sensitive)? Does the surface serve non-English users? In-house panels with specialised experts are expensive and slow to scale; vendor pools claim coverage but the depth varies widely across vendors and languages.
- **Operator control.** How fast can the team rewrite the rubric, change the UI, adjust the sampling? In-house pools iterate within a day; vendor pools iterate on the change-order cadence (often weeks).
- **Exit cost.** If the vendor relationship ends — vendor acquires, vendor drops the product, vendor's quality degrades, DPA expires — what does the eval team lose? Labellers walk with the vendor; rubric-trained tacit knowledge walks; the next vendor requires a re-onboarding cycle.
- **Rubric iteration speed.** How fast can the rubric change? In-house: within a sprint. Vendor: contingent on the vendor's change-management posture — typically two to four weeks for a non-trivial revision, with re-training included.

No single path optimises all seven. The vendor path dominates on cost at scale and on steady-state throughput. The in-house path dominates on iteration speed, data-residency control, and exit cost. The hybrid path trades a little on each axis for a lot on the aggregate.

### The vendor landscape as shape templates

Several vendors operate in the LLM-labelling space. This section reads them as *shape templates* — the capabilities to look for, the posture to expect — rather than as endorsements. Any specific vendor's offering evolves; the categories are what to compare across.

| Vendor | Shape template | Typical product-team fit |
|---|---|---|
| **Scale AI** | High-volume, cross-industry, strong on specialised pools (defense, autonomous vehicles, LLM eval). Enterprise contracting; dedicated program managers. <!-- needs-research: re-verify Scale AI's current LLM-feedback offerings and pricing posture at https://scale.com/ --> | Large teams running steady-state labelling at scale with enterprise contracting and support for specialised verticals. |
| **Surge AI** | LLM-first; marketed on quality and reviewer vetting; smaller, more curated pool. <!-- needs-research: re-verify Surge AI's current vetting process and offerings at https://www.surgehq.ai/ --> | Teams prioritising label quality on nuanced rubrics; moderate volume. |
| **Labelbox** | Platform + marketplace; teams can bring their own labelling pool or use Labelbox's; strong integration surface. <!-- needs-research: re-verify Labelbox's current platform capabilities at https://labelbox.com/ --> | Teams wanting the vendor as a *platform* (bring-your-own pool) rather than a pure labelling service. |
| **Appen** | Established industry player; global crowd; broad vertical and language coverage; traditional data-labelling scale. <!-- needs-research: re-verify Appen's current services at https://appen.com/ --> | Multi-lingual surfaces; traditional labelling tasks alongside LLM feedback. |
| **iMerit** | Managed workforce with vertical specialisation (medical, geospatial, autonomy); enterprise contracting. <!-- needs-research: re-verify iMerit's current vertical specialisations at https://imerit.net/ --> | Domain-heavy rubrics where vertical expertise is the bottleneck. |
| **Toloka** | Crowd-based platform with access to a global pool; can be run self-serve or managed. <!-- needs-research: re-verify Toloka's current platform and data-residency posture at https://toloka.ai/ --> | Teams wanting direct crowd access with per-task configuration; data-residency considerations needed. |
| **Prolific** | Research-oriented participant pool; strong on participant vetting and demographic recruitment; widely used in academia and increasingly in LLM-eval. <!-- needs-research: re-verify Prolific's current LLM-eval use-cases at https://www.prolific.com/ --> | Teams needing demographic-specific feedback or academic-style study recruitment; smaller volume, higher participant quality. |

Three properties to probe when evaluating any vendor:

- **Labeller vetting.** How the vendor selects and validates labellers; what pass-rate the vendor targets on calibration tasks; how retraining cycles run. The `model-evaluation-engineer` peer audits this directly; the module consumes the audit report.
- **IAA reporting back to the customer.** Does the vendor compute kappa / alpha on the shipped labels and surface it? Is the sample-size CI included? Unchecked vendor labels at κ 0.5 on a rubric that required κ 0.8 wreck the mod-104 calibration; the module cannot catch this without the vendor's own IAA (or without running its own on a vendor-labelled subset — chapter 04).
- **Data-residency and contracting posture.** DPA, sub-processor list, regional pool availability, PII-handling certifications (SOC 2, ISO 27001, HIPAA where applicable). Pretend these are optional at the team's peril.

### The in-house build-cost

The in-house path is often undercounted. A product-altitude estimate for a steady-state in-house panel:

- **People.** A core pool of 2 – 4 reviewers plus a senior adjudicator. Fully-loaded cost per reviewer (salary + benefits + overhead) runs $70k – $180k / year depending on domain and region. A specialised panel (medical, legal, security) is at the high end.
- **Management overhead.** 10 – 20 % of the pool's time absorbed in rubric meetings, training, QA, IAA review, scheduling. A senior adjudicator spends 30 – 50 % of their time on management.
- **Workbench.** Argilla self-hosted or Label Studio self-hosted + Enterprise-equivalent SSO + audit trail. Ballpark infra cost is low compared to labour.
- **Training and onboarding.** First month of a new reviewer's labels is at reduced throughput (and will fail IAA until the κ against the panel exceeds the threshold, chapter 04). Budget 4 – 6 weeks of ramp per hire.

The implicit advantage: every artefact the module produces (rubric, calibration set, gold rotation, triage rules) is co-produced with the labelling labour. Rubric iteration is same-day. Exit cost is near-zero (there is no vendor to exit). Data-residency is a non-issue (the labour is where the enterprise is).

The implicit disadvantage: the pool has a throughput ceiling, and bursty queues (a mod-108 safety-eval sweep, a mod-107 drift-triggered re-label) blow it. Scaling in-house is weeks-to-months; vendors scale in days.

### The hybrid pattern most product teams converge to

The hybrid pattern is the workhorse for a mature program. Two configurations are common.

- **In-house experts own the rubric and the gold; vendor pool runs the high-volume labelling.** Chapter 05's rotation is signed off in-house; chapter 04's IAA is monitored on a vendor-labelled subset co-labelled by an in-house reviewer; the bulk of the chapter 02 workbench throughput is vendor. The in-house panel is 1 – 2 experts; the vendor pool is 10 – 50 labellers.
- **In-house primary reviewers; vendor pool for burst capacity.** The steady state is in-house; the burst (a safety sweep, a re-label after a rubric revision, a one-off gold rotation at an unusual size) spills to a vendor. The vendor contract is a flat retainer plus per-row pricing; the pool is smaller but on-call.

The hybrid pattern makes three operational commitments:

- **Shared schema.** The vendor labels against the same chapter-02 schema the in-house pool uses. No vendor-specific columns, no translation layer. Vendors that cannot ingest the shared schema cleanly get filtered early.
- **Vendor-labelled rows are co-labelled on a sample.** A 5 – 10 % overlap with the in-house pool runs in parallel so that chapter 04's IAA can be computed on the overlap. Vendors whose κ against the in-house pool slips below the rubric's threshold are flagged, re-trained, or rotated out.
- **The gold rotation is in-house-only.** Chapter 05's rotation signs with the rubric owner or the policy owner; the vendor pool does not approve rotations. This keeps the strategic artefact with the eval team.

### The delegation to the `model-evaluation-engineer` peer

The module stops at the operational decision; the peer audits the vendor methodology. Three handoff points are common.

- **Pre-contract audit.** Before signing with a new vendor, the peer reviews the vendor's labeller-selection process, calibration-task design, IAA reporting shape, and retraining cadence. The audit returns a pass / fail against the methodology bar the peer maintains.
- **Mid-contract quality review.** On a quarterly cadence, the peer samples the vendor's labelled output, runs an independent IAA against a held-out in-house sample, and reports the result. If the vendor's reported IAA and the peer's measured IAA disagree by more than a tolerance, the peer flags the vendor's reporting.
- **Rubric-change methodology review.** When the module's rubric changes materially, the peer reviews whether the vendor's re-training cycle meets the methodology bar for re-measurement. A rubric change without the peer's sign-off on the re-training plan risks shipping a vendor pool that was trained against the old rubric.

The module runs the queue, picks the vendor, negotiates the SLA, and consumes the labels. The peer tells the module whether the vendor is doing its job to a standard the rest of the org accepts.

### Data-residency and PII — the hard constraints

Two cases where the vendor option is closed by constraint rather than by preference.

- **The rows are PII-bearing and cannot leave the enterprise boundary under any vendor DPA.** The module labels in-house; no vendor is an option for this surface. The module's labour model for this surface is fully in-house, regardless of the cost comparison.
- **The surface serves regulated verticals with jurisdictional labelling requirements.** EU personal-data rows cannot be labelled by a non-EU pool. Some vendors offer EU-only pools at a premium; others do not. The module's choice collapses to "vendor with EU pool" or "in-house in the EU"; the global-vendor option is foreclosed.

Three disciplines make the data-residency posture defensible:

- **Row-level residency tagging.** Every row carries a residency tag (`us_only`, `eu_only`, `on_prem_only`); the import job to the workbench routes to a residency-compatible pool. A row with `eu_only` cannot land in a global vendor pool even if the pool is nominally cheaper.
- **DPA and sub-processor review before pilot.** Pilot labelling with a vendor before the DPA is signed leaks data into an unauthorised processor. Run the DPA review first, then the pilot.
- **The hybrid falls back to in-house when residency closes the vendor option.** For an EU-only surface at a team without an EU vendor contract, the hybrid pattern collapses to in-house; the cost picture reads accordingly.

### What this module does not decide

- **The procurement process.** Vendor selection, RFP, contract negotiation, DPA review, renewal cadence are a program-ops function. The module supplies the technical trade-off memo (the chapter-06 output artefact); procurement runs the commercial process.
- **The vendor's internal operations.** The vendor's own quality-control process, its labeller-pay model, its retraining cadence, its sub-contracting posture — all are the vendor's responsibility (and the peer's audit target). The module reads the audit; it does not run it.
- **The compliance posture of the enterprise.** SOC 2, ISO 27001, HIPAA, GDPR, PCI DSS and their local equivalents are enterprise-wide artefacts. The module's labour model has to fit them; it does not define them.

## Summary

- **Seven trade-off axes** govern the pick: cost per label, latency to first label, data-residency / PII posture, vertical / language coverage, operator control, exit cost, rubric iteration speed. No single path optimises all seven.
- **Vendors read as shape templates** — Scale AI, Surge AI, Labelbox, Appen, iMerit, Toloka, Prolific each have a posture to compare against. The actual vendor pick evolves with the market; the probe questions (labeller vetting, IAA reporting, data-residency posture) are what remains stable.
- **In-house build-cost** is 2 – 4 reviewers + a senior adjudicator at $70k – $180k fully-loaded per reviewer, plus 10 – 20 % management overhead and a 4 – 6 week ramp per hire. Upside: same-day rubric iteration, near-zero exit cost, data-residency as a non-issue. Downside: a throughput ceiling that bursty queues blow.
- **Hybrid is the workhorse**: in-house experts own the rubric and the gold; vendor pool runs the high-volume labelling; a 5 – 10 % co-labelled overlap maintains the IAA signal. Alternative hybrid: in-house steady state, vendor on-call for bursts.
- **Peer delegation** covers vendor-methodology audits (pre-contract, mid-contract quarterly, rubric-change re-training review). The module picks the vendor; the peer audits whether the vendor is doing its job.
- **Data-residency and PII** are hard constraints — some surfaces foreclose the vendor option entirely; row-level residency tagging and DPA review before pilot are non-negotiable.
- **The module does not decide** procurement, the vendor's internal operations, or the enterprise compliance posture. It writes the technical trade-off memo and consumes the audit.

The module ends here. The next step in the curriculum is mod-110's eval data platform, which stores every artefact this module produced — reviewed rows, triage items, IAA reports, gold snapshots, and the vendor-vs-in-house decision memo — as first-class tables with full lineage.
