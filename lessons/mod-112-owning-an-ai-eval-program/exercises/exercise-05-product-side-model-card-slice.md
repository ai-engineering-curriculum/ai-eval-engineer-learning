# exercise-05: Product Side Model Card Slice

**Estimated effort:** 2 hours

## Objective

Author the chapter 06 **card-slice generator** for one app surface family — the deterministic composer that reads the release-gate history, the trade-off reports, the safety reports, the investment ledger, the framework mapping, and the model-eval-eng peer's capability profile, and emits a timestamped eight-section Markdown slice at a well-known path the governance peer's card-assembly pipeline reads. Then produce the first slice for the family, land it on the schedule, and run the first quarterly slice-diff review with the governance peer (or a stand-in).

By the end of the exercise the eval program's contribution to the org-wide model / system card is **generated, not authored**. The chapter 06 symptoms — the governance peer paraphrasing eval dashboards, the card stale within weeks of publication — are closed; the slice is reproducible, timestamped, source-hash-stamped on every number, and audience-tagged so the peer can compose cards for internal risk review, third-party procurement, regulatory submission, and public release without rewrites.

## Prerequisites

- Chapters 01, 02, 03, 05, 06, and 07 of this module.
- Exercise 01 — the release-gate history log is one of the slice's inputs.
- Exercise 02 — the governance peer contracts (`ai-evaluation-engineer` L25 and `ai-governance-analyst`) define the slice's consumer contract.
- Exercise 04 — the investment ledger provides §5 (limitations) and §7 (incidents).
- Mod-108 (safety) report per surface in the family — the slice's §4 reads these.
- Mod-110 (platform slice) tables (`eval_runs`, `eval_results`, `pricing_snapshots`, `capability_profiles`) — the generator reads from these. Scaled-down substitutes (CSV + YAML) are acceptable if mod-110 is in bootstrap.
- Mod-111 (cost / latency / quality trade-off) reports per shipped candidate — the slice's §2 reads these.
- Access to the governance peer (or a stand-in) who can review the first slice and confirm the "folded in without rewrites" contract holds.
- A model / system card template to target. If the org has one, use it; if not, adapt the Mitchell 2019 shape or a provider-published shape (Anthropic / OpenAI / Google DeepMind / Meta — see `resources.md`).
- Python 3.11+; `pyyaml`; `jinja2` for the slice template; a hashing library (`hashlib`) for source-hash provenance.

## Set-up

1. Create the card-slice directory:

   ```
   programs/card_slices/<family>/
   ├── generate.py                             # the deterministic generator
   ├── template.md.j2                          # the Jinja2 template for the slice
   ├── inputs.yaml                             # paths + versions of every input artefact
   ├── slice_schema.json                       # schema for the generator's output (§ reproducibility)
   ├── sections/
   │   ├── 01-intended-use.md                  # family-level product framing (eval adds hook)
   │   ├── 02-evaluation-results.py            # per-surface rubric scores vs. thresholds
   │   ├── 03-model-capability-profile.py      # by reference to model-eval-eng peer
   │   ├── 04-safety-evaluation-results.py     # from mod-108
   │   ├── 05-known-limitations.py             # from investment ledger
   │   ├── 06-monitoring-and-change-process.py # from release-gate policy
   │   ├── 07-recent-incident-summary.py       # from investment ledger
   │   └── 08-reproducibility-artefacts.py     # pointers + hashes
   ├── output/
   │   └── <YYYY-QQ>/
   │       ├── slice.md                        # the generated slice (committed)
   │       ├── slice.json                      # machine-readable form (committed)
   │       ├── sources.lock                    # hashes of every input source
   │       └── diff-vs-previous.md             # auto-generated diff against prior cycle
   ├── audiences/
   │   ├── internal_risk.yaml                  # tag → sections subset
   │   ├── third_party_procurement.yaml
   │   ├── regulatory.yaml
   │   └── public.yaml
   ├── validate/
   │   ├── validate_schema.py                  # slice conforms to slice_schema.json
   │   ├── validate_sources.py                 # every number traces to a source + hash
   │   ├── validate_determinism.py             # same inputs → same bytes
   │   ├── validate_audience_tags.py           # every section has an audience-tag list
   │   └── validate_handoff_path.py            # output lands at the path the peer reads
   ├── review/
   │   └── quarterly-slice-diff-review.md
   ├── history/
   │   ├── 2026-Q4-initial.md
   │   └── reviews/
   │       └── .gitkeep
   └── README.md
   ```

2. Fill `inputs.yaml` with every input source the generator reads. One entry per source with path, version pointer, and read-shape:

   ```yaml
   inputs:
     release_gate_history:
       path: programs/gates/<family>/history/*.json
       read: latest_N_releases_per_surface
       N: 5
     tradeoff_reports:
       path: eval/tradeoff/report/output/*/report.json
       read: latest_per_surface_per_candidate
     safety_reports:
       path: eval/safety/output/*/report.json
       read: latest_per_surface
     investment_ledger:
       path: programs/investment/ledger.yaml
       filter: family == "<family>"
     framework_mapping:
       path: programs/framework_mapping/<family>.yaml
       read: full
     capability_profiles:
       path: mod-110://capability_profiles       # or a CSV substitute
       filter: model_snapshot in production_on_family
     mod110_substitute:
       path: eval/mod-110-substitute/*.csv       # if mod-110 is bootstrap
       enabled: <true|false>
   ```

3. Author `template.md.j2` with the chapter 06 eight-section shape. Every section header includes an `audience_tags` front-matter line and every number uses a Jinja filter that embeds the source hash:

   ```jinja
   # Card Slice — {{ family }} — {{ cycle }}

   > Generated at {{ generated_at }}; sources.lock hash: {{ sources_hash }}
   > Audience tags: see § per section front-matter.

   ## § 1 Intended use
   <!-- audience: internal, third_party, regulatory, public -->
   {{ sections.intended_use }}

   ## § 2 Evaluation results per surface
   <!-- audience: internal, regulatory -->
   {% for surface in sections.evaluation_results.surfaces %}
   ### {{ surface.name }}
   Release: {{ surface.release_id }} ({{ surface.release_date }})
   Pre-registered rule: `{{ surface.rule | source('tradeoff_report') }}`
   {{ surface.rubric_table_md }}
   {% endfor %}

   ## § 3 Model capability profile
   <!-- audience: internal, third_party, regulatory -->
   {{ sections.capability_profile.by_reference_summary }}
   Peer-published source: {{ sections.capability_profile.peer_source_url }}
   Snapshot hash: {{ sections.capability_profile.hash }}

   ## § 4 Safety evaluation results
   <!-- audience: internal, third_party, regulatory, public -->
   {{ sections.safety.summary_md }}

   ## § 5 Known limitations and mitigations
   <!-- audience: internal, third_party, regulatory, public -->
   {{ sections.limitations.md_table }}

   ## § 6 Monitoring and change process
   <!-- audience: internal, regulatory -->
   {{ sections.monitoring.summary_md }}

   ## § 7 Recent incident summary
   <!-- audience: internal, regulatory -->
   {{ sections.incidents.md_table }}

   ## § 8 Reproducibility artefacts
   <!-- audience: internal, regulatory -->
   {{ sections.reproducibility.md }}

   ---
   Slice hash: {{ slice_hash }}
   Signers: {{ signers | join(', ') }}
   ```

4. Author one `sections/<n>-*.py` module per section. Each module exports a function `render(inputs: Inputs) -> SectionDict` that reads the inputs, composes the section, and returns a dict the template renders. The modules are the generator's workhorse; they are what the schema and determinism validators exercise.

5. Fill `audiences/*.yaml` with the per-audience section tag maps:

   ```yaml
   # audiences/public.yaml
   name: public
   sections_included: [1, 4, 5]
   redactions:
     - rule: strip_cohort_specific_numbers_from_section_4
     - rule: strip_incident_ids_from_section_5
   ```

6. Register the slice's well-known path in the governance-peer contract (exercise 02's `contracts/ai-evaluation-engineer-governance.yaml` and / or `ai-governance-analyst.yaml`). The slice lands at `programs/card_slices/<family>/output/<cycle>/slice.md`; the peer's card-assembly pipeline reads that path. If the contract does not already name the path, update the contract in the same PR.

## Requirements

Produce a PR against your eval-program repo that adds:

1. **`generate.py`** — the deterministic generator; CLI `python generate.py --family <family> --cycle 2026-Q4`; reads `inputs.yaml`; dispatches to `sections/*.py`; renders via `template.md.j2`; writes `output/<cycle>/slice.md`, `slice.json`, `sources.lock`, and `diff-vs-previous.md`.
2. **`template.md.j2`** — the Jinja2 template with the eight-section shape; audience-tag front-matter per section; source-hash embedded on every number.
3. **`inputs.yaml`** — the per-family input source registry; every input has a path, a read shape, and (where applicable) a version filter.
4. **`slice_schema.json`** — JSON Schema for `slice.json`; eight sections required; audience-tags required per section; `sources_hash`, `generated_at`, `signers` required at top level.
5. **`sections/*.py`** — eight section modules; each exports `render()` with a deterministic implementation; each module is unit-tested with a fixture input.
6. **`validate/validate_schema.py`** — `slice.json` validates against `slice_schema.json`; failure names the missing field.
7. **`validate/validate_sources.py`** — every number in `slice.json` has an accompanying `source` key with a non-empty hash; no number appears without a source citation (the chapter 06 "slice is a dashboard link" anti-pattern catch).
8. **`validate/validate_determinism.py`** — runs the generator twice against the same inputs; diffs the outputs; passes only if the two outputs are byte-identical. Catches a hidden `datetime.now()` or `random.random()` in a section module.
9. **`validate/validate_audience_tags.py`** — every section has an `audience_tags` list; every tag is one of the four known audiences (`internal`, `third_party`, `regulatory`, `public`); a section with no public tag that contains PII-adjacent numbers is flagged.
10. **`validate/validate_handoff_path.py`** — reads the governance-peer contract from `programs/delegation/contracts/`; confirms the `card_slice` artefact's `lands_in` path matches `output/<cycle>/slice.md`; the chapter 06 "well-known path" contract.
11. **`output/<cycle>/slice.md`** — the first generated slice for the family; eight sections populated; signed by the eval program owner and the family's product + engineering lead.
12. **`output/<cycle>/slice.json`** — the machine-readable form.
13. **`output/<cycle>/sources.lock`** — the hash manifest of every input source used; the lock file is what makes the slice reproducible when the inputs change.
14. **`output/<cycle>/diff-vs-previous.md`** — the auto-generated diff; for the first cycle, "no previous cycle" is a valid output and the file documents the baseline.
15. **`audiences/*.yaml`** — four audience tag maps (`internal_risk`, `third_party_procurement`, `regulatory`, `public`); each with the included-sections list and any redaction rules.
16. **`review/quarterly-slice-diff-review.md`** — the standing agenda per chapter 06 § "Update cadence": what changed per section; sections still fit-for-purpose; framework mapping reflected accurately; drift between slice and peer's card. The first review's notes land in `history/reviews/`.
17. **CI integration** — the five validators run on every PR touching `programs/card_slices/`; the determinism validator runs the generator twice as part of CI.
18. **Governance-peer contract update** — the `card_slice` artefact's `lands_in` path and consumer binding are current in the contract; if the path moved, both contract and generator are updated in the same PR.
19. **`history/2026-Q4-initial.md`** — the authoring note: which inputs are live vs. bootstrap substitutes; which audiences are configured today; the first quarterly-slice-diff-review calendar entry; any `<!-- needs-research: ... -->` markers for template evolution.
20. **`README.md`** — the directory's self-standing overview: how to run the generator, how to interpret the audience tags, how to run the quarterly review, how to add a new input source.
21. **One first-run quarterly-slice-diff review** — scheduled with the governance peer (or stand-in) within 90 days of the authoring PR; notes land in `history/reviews/2026-Q4.md` after the session.

## Starter guidance

- **Never author the slice by hand.** Chapter 06 is explicit: the slice is **generated**. The temptation is to open `slice.md` and tweak a sentence. Resist it. Every tweak drifts from the source numbers; every drift breaks the "zero rewrites" contract with the governance peer. If the slice needs a different sentence, the template or the section module changes, not the output.
- **Determinism is a contract, not a nice-to-have.** The chapter 06 "reproducibility artefacts" section depends on it. Same inputs → same bytes. A `datetime.now()` sneaks in through a logging line; a dict iteration order flips; a non-sorted set changes order. The determinism validator catches these — run it before every commit.
- **Every number carries its source hash.** The template filter `| source('tradeoff_report')` is the discipline. A number without a source hash is a drift waiting to happen; the `validate_sources.py` validator flags it. The source hash lets a reader six months later confirm the number came from `report.json` version X.
- **Source by reference, not by re-derivation.** Chapter 06 § 3 is explicit for the capability profile — the model-eval-eng peer owns it; the slice links to it, does not re-run it. The same discipline applies to every section: the slice *composes*; it does not *re-produce*. If a section is tempted to score a new rubric, that work belongs in the mod-104 rubric runner, not in the generator.
- **Audience tags are per section, not per card.** The slice is authored once with every section tagged; the governance peer composes per-audience cards by picking section subsets. A public card excludes §2 (per-cohort numbers) and §7 (incident ids); a regulatory card includes everything plus the chapter 07 framework-mapping annotations. The tags are the mechanism.
- **The well-known path matters more than you think.** The chapter 06 "handed cleanly to the governance peer" contract has four requirements; well-known path is first. If the peer's pipeline reads `programs/card_slices/<family>/output/<cycle>/slice.md`, the generator writes exactly that path. Changing the path is a contract update (exercise 02's `ai-evaluation-engineer-governance.yaml`), not a unilateral eval-side change.
- **Diff-vs-previous is what makes the quarterly review useful.** Without the diff, the review is "walk the whole slice again"; with the diff, the review is "what changed and why." The auto-diff names the sections and the numbers that moved; the review names the significance.
- **Out-of-cadence regeneration is cheap; make it the default on P0 / P1 close.** When an investment-ledger row transitions to `landed`, the next slice regeneration (chapter 06 § "Update cadence") picks up the new §5 and §7 rows. For this exercise, add a Make target or a watch-script that regenerates the slice when `programs/investment/ledger.yaml` changes.
- **Mitchell 2019 is a shape, not a template.** The chapter 06 reference to "Model Cards for Model Reporting" is a shape — nine sections, specific framing. Your org's template is an extension. Author the generator against your org's template; the Mitchell paper is a sanity check for completeness.
- **Four audiences is a floor, not a ceiling.** `internal`, `third_party`, `regulatory`, `public` cover the chapter 06 shape. Your org may have additional ones (board, insurance, external red-team partners). Add tags; keep the generator agnostic to the number of audiences.
- **The first quarterly review is a stress test of the "zero rewrites" contract.** When the governance peer folds the slice into the org-wide card, watch for rewrites — a sentence rephrased, a number restated, a limitation reworded. Every rewrite is a signal: either the slice is wrong for the card (fix the generator) or the peer is doing work the slice should do (expand the slice). The review notes record which.
- **A `<!-- needs-research: ... -->` in the template is fine.** The chapter 06 "needs-research" markers for the Mitchell shape and provider-published cards are expected to drift; mark them, move on, and the next quarterly review refreshes.

## Acceptance criteria

You are done when:

- `generate.py` runs end-to-end against the family's inputs; produces `output/<cycle>/slice.md`, `slice.json`, `sources.lock`, `diff-vs-previous.md`.
- Two consecutive runs against identical inputs produce byte-identical outputs; `validate_determinism.py` passes.
- Every number in `slice.json` has a `source` field with a non-empty hash; `validate_sources.py` passes.
- Eight sections are populated; each has an `audience_tags` list; `validate_audience_tags.py` passes.
- The slice lands at the path the governance-peer contract names; `validate_handoff_path.py` passes.
- `output/<cycle>/slice.md` is signed by the eval program owner and the family's product + engineering lead.
- Four audience configurations are present; each produces a different section subset when applied.
- The quarterly-slice-diff-review calendar entry exists and the first review is scheduled with the governance peer (or stand-in) within 90 days.
- CI runs the five validators on every PR touching `programs/card_slices/`; a deliberate break (strip a source hash, author a §2 line by hand, move the output path without updating the contract) fails CI.
- `history/2026-Q4-initial.md` names which inputs are bootstrap substitutes and names the first review date.
- A governance peer (or stand-in) opens `output/<cycle>/slice.md` and can fold it into a per-audience card template without rewrites — if any rewrites are needed, they are logged as review items in `history/reviews/2026-Q4.md`.

## Stretch goals

- **Per-audience render.** Extend `generate.py` with a `--audience <tag>` flag that applies the audience's section subset and redaction rules and emits a per-audience slice (`slice-public.md`, `slice-regulatory.md`). The chapter 06 subsettable-sections pattern becomes self-serve.
- **Slice-diff bot.** On every slice regeneration (monthly default, out-of-cadence on P0/P1 close or gate refresh), post a summary to the eval-program channel highlighting the changed sections with the specific number moves. Reduces the "nobody noticed §5 changed" surface area.
- **Framework-mapping annotations.** When the regulatory audience is rendered, auto-insert the chapter 07 framework-mapping citations next to each section — "§6 Monitoring and change process (evidences NIST AI RMF MANAGE 4.1; EU AI Act Article 72; ISO/IEC 42001 Clause 9.1)". The regulatory audience's slice becomes an audit-ready brief.
- **Historic slice browser.** A small directory-listing over `output/*` with timestamps, cycle labels, and diff-summary snippets. The eval team's institutional memory over card evolution is browsable.
- **Signed slice attestations.** Replace the signer dates with cryptographic signatures (Sigstore, in-toto, DSSE). Every slice is a signed attestation; the governance peer can verify authenticity when folding into the card.
- **Peer-side consumption test.** A tiny shim that pretends to be the governance peer's card-assembly pipeline — reads the well-known path, folds the slice into a stub card template, flags any required field that is missing. The shim is what the chapter 06 contract tests against before the real peer picks it up.
- **Public-card diff vs. provider cards.** Pull the current published Anthropic / OpenAI / Google DeepMind system cards (`resources.md` has pointers) and diff section-by-section against the public audience's slice. The diff surfaces sections your card lacks that a procurement reviewer will expect.

## What this exercise does *not* cover

You are not authoring the release-gate architecture (exercise 01), the delegation contracts (exercise 02), the build-vs-buy matrix (exercise 03), or the investment-ledger walkthrough (exercise 04). You are shipping the *card-slice generator* for one app surface family — the chapter 06 artefact that hands the eval program's evidence to the governance peer with zero rewrites and makes the org-wide model / system card live-updating rather than stale-at-publication.
