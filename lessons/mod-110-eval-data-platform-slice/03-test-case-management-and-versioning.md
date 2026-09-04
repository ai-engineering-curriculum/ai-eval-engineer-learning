# Test-Case Management and Versioning: Letting Product Teams Contribute Without Touching the Harness

## Motivation

The single most common failure mode of an eval program at year one is not the rubric, the judge, or the store — it is the *contribution flow*. Product teams have the ground-truth knowledge of what "correct" looks like; the eval team has the machinery. If the machinery is the only surface for adding a case, three things happen:

- The product team stops adding cases. The eval set ossifies against the shape of usage the harness first saw.
- The product team keeps a private CSV of "cases we care about," runs their own ad-hoc checks, and the results never land in the shared store. The scorecard the org reviews and the scorecard the team actually trusts diverge.
- When a case is added, it lands as a whole new suite ("team-A extra cases v1"), the hash of the *existing* golden set does not move, and nobody notices that the golden set no longer covers team A's surface.

The chapter's job is to design a contribution surface that is safe for a non-engineer product-team member to use, that keeps every case attributable to an owner, that versions the eval set with the same rigor the codebase gets, and that ties the resulting `eval_set_hash` (chapter 02) back to a specific, auditable set of contributions.

The contract is small: **product teams contribute cases; the platform normalises, validates, hashes, versions, and stores them; runs read a specific `eval_set_hash` and never a mutable "latest."**

## Core concepts

### Two contribution surfaces

Two shapes cover most orgs. Pick one, or offer both — but treat them as first-class, not as a "temporary workflow until we build the UI." Both should exist for years.

**Surface A: eval-set-as-code (YAML in repo, PR review, CI validation).**

- The eval set is a directory of YAML files in a Git repo. One file per case, or one file per group of cases with a shared header.
- Contributors open a PR against the repo. The CI runs schema validation, tag validation, ownership validation, and a plane-side `plan --dry-run` that would fire the case through the runner without paying the judge.
- The PR reviewer is the *owning team's* engineer or lead, plus (for cross-team-visible suites) the eval team.
- On merge, a CI job computes the new `eval_set_hash`, bumps the semver, records the contributor, tags, and case_ids in the `eval_cases` table, and inserts a new row in `eval_sets` with the new hash and `supersedes` pointer.

Strengths: fits engineering workflows the org already has; free code review, free audit trail via Git; four-eyes review by default. Weakness: a non-engineer needs help opening a PR.

**Surface B: eval-set-as-database (UI plus API, four-eyes review).**

- The plane exposes a small internal UI. A product-team member logs in, picks the suite, clicks "add case," fills in a form (inputs, expected outputs, tags, description of the failure mode this case catches), and submits.
- The submission lands as a `proposed` case row. A second person (owner-team lead or eval team) reviews and approves, at which point the case joins the suite and the `eval_set_hash` bumps.
- Every action is audited: `submitted_by`, `submitted_at`, `reviewed_by`, `reviewed_at`, `approved_by`, `approval_reason`.

Strengths: safe for non-engineers; low friction; the platform enforces the four-eyes rule mechanically. Weakness: harder to bulk-edit; the audit lives in the platform, not in Git.

The choice depends on the product team's shape:

| Product team shape | Surface fit |
|---|---|
| Engineering-heavy, uses Git daily | A (eval-set-as-code) |
| Product / support / T&S with rare Git use | B (eval-set-as-database) |
| Mixed org | Both — most contributions land through B; large batches or cross-cutting refactors land through A |

The two surfaces write to the **same schema** — `eval_cases` and `eval_sets` from chapter 02. A case does not care which surface added it; the `contributor` column just records the person. This is the property that keeps the two surfaces from turning into two different systems.

### The case-id contract

Every case has a stable `case_id`. The rules:

- **Stable across suite versions.** Adding a case bumps the `eval_set_hash`; the *existing* cases keep their ids. This is what lets a longitudinal query — "how did case `CASE-billing-refund-2026-03-12` score across the last twelve golden runs?" — answer at all.
- **Meaningful.** Not `CASE-000042`. The id encodes the suite prefix, the failure mode (or subject), and either the contribution date or a monotonic counter — `CASE-support-billing-refund-2026-03-12`, `CASE-safety-jailbreak-encoded-072`. Human-readable in Git blame, in the runbook, in the dashboard.
- **Never reused.** A deleted case is soft-deleted (`retired_at` column) — its id never comes back. If the underlying content is replaced by a corrected version, the corrected version gets a new id and the old id gets a `superseded_by` pointer.
- **Hashed separately from the case body.** The `case_hash` (SHA-256 of the canonicalised body) is what changes when the case content changes; the `case_id` is stable. A case whose expected output was corrected keeps its `case_id` and gets a new `case_hash`, and this change bumps the `eval_set_hash` of every suite it belongs to.

Case ids are the join key everywhere downstream — the mod-107 dashboard filters by them, the mod-108 report cites them, the mod-111 cost breakdown per-case reads them. Get this contract right on day one; it is expensive to fix later.

### The `eval_set_hash` in detail

Chapter 02 said `eval_set_hash` is a SHA-256 over the canonical serialisation of the eval-set contents. The canonicalisation:

```
canonical_form = json.dumps(
    sorted(
        [{"case_id": c.case_id, "case_hash": c.case_hash} for c in cases],
        key=lambda x: x["case_id"],
    ),
    separators=(",", ":"),
    ensure_ascii=False,
)
eval_set_hash = "sha256:" + hashlib.sha256(canonical_form.encode("utf-8")).hexdigest()
```

Properties this gives you:

- **Order-independent.** Two contributions that add the same case set in different orders produce the same hash.
- **Content-sensitive.** Editing a single case's body changes the case's `case_hash`, which changes the eval-set hash.
- **Membership-sensitive.** Adding, removing, or retiring a case changes the hash.
- **Comment / whitespace-insensitive.** The canonicalisation happens on the *body*; formatting changes in the YAML file are invisible to the hash. This is intentional — a whitespace-only PR should not bump the hash and force a re-run of every gate.

Two rules the exercise enforces:

- **A run always names a specific `eval_set_hash`.** Not `eval_set_id="support_bot_golden"` with an implicit "whatever the current version is." The `EvalRunSpec` resolves the id to a hash *at spec time* and stores the hash in the spec. This is what makes the mod-106 replay bundle immutable: the replay runs against a *specific* golden set, not against whichever golden set happens to be current at replay time.
- **The current-version pointer is a separate table.** `eval_set_current` maps `suite -> eval_set_id -> eval_set_hash`. New runs default to the current pointer; old runs keep their historic hash. Bumping the pointer is an explicit operation with an audit row.

### Suite structure and the three subset roles

Every product surface benefits from three named subsets of its eval set. The suite structure the chapter recommends:

- **Golden.** The reference set. Small (30 – 500 cases), curated, high-signal per case, changes rarely, every change is peer-reviewed at high care. The regression gate (mod-106) runs against this set on every PR that touches the surface. Score movements here are load-bearing.
- **Regression.** Cases that *have* regressed at some point in the past. Every P2+ incident adds at least one case ("the retrieval-index rebuild broke this query," "the prompt refactor broke this multi-turn"). Grows monotonically. Ensures we do not silently reintroduce a fixed bug. The regression gate runs it as a separate scorecard so a new regression is loud.
- **Rot-check.** Cases sampled from real production traffic (via the mod-107 online-loop sampler) that failed a spot-check at some point. Rebuilt on a monthly cadence. Catches distribution drift the golden set never saw. Runs in a nightly job, not on the PR gate.

Each subset is a separate `eval_set` row with its own hash and version. A single suite (`support_bot`) has three eval-set ids: `support_bot_golden`, `support_bot_regression`, `support_bot_rot_check_2026_08`. The plane joins them at query time.

### Tags: the scoring axis for slice-level reporting

Every case has a **tag set**. Tags are the axis the reporting slices on — chapter 06 of mod-107's cross-persona dashboard reads them, mod-108's OWASP mapping reads them, mod-111's per-tenant cost breakdown reads them.

Tag vocabulary rules:

- **Controlled vocabulary.** Not free-form. A CI check rejects a case with a tag not in the current tag catalog. The catalog is a versioned YAML in the same repo (or a plane table with the same four-eyes review rule).
- **Multiple axes.** `feature:{billing, onboarding, ...}` (product feature), `capability:{extraction, retrieval, reasoning, generation, tool_call, safety_refusal, ...}` (LLM capability), `criticality:{must_never_regress, target, aspiration}` (SLO tier), `tenant_visible:{yes, no}` (customer-facing).
- **Additive.** A case can carry many tags. Filtering an axis is the primary UI in every downstream report.
- **Migratable.** A tag rename is a first-class operation — script the migration, bump the tag-catalog version, re-tag every affected case, audit the change. Not a spreadsheet find-and-replace.

Ownership: every case has an `owner_team`. The tag catalog has a table mapping tag values to owning teams (or "eval-team-shared" for the cross-cutting ones). When a case fails, the runbook link routes by tag → owner.

### Semver on the suite

Every eval set is semver-versioned. The interpretation:

- **Patch bump.** Additive-only change: added cases, fixed typos in a case's `description` field, added tags. Existing cases unchanged. Downstream runs that pinned a minor can safely opt in.
- **Minor bump.** Backward-compatible content change: renamed a tag, corrected a case's expected output, retired a case (with `superseded_by`). Existing runs' hashes are unaffected because they name the old `eval_set_hash`; new runs default to the new minor.
- **Major bump.** Backward-incompatible shape change: the case body schema changed (new required field, removed field), the metric family changed (e.g., faithfulness moves from a numeric 0 – 1 to a categorical PASS / FAIL). Every runner adapter reads the version; a runner that has not been updated to the new major refuses to run.

The semver is a *label* on top of the content hash. The hash is what the reproducibility check trusts; the semver is what the runners and the humans co-ordinate on.

### The change-review contract

A PR that adds cases is a change to the eval scorecard the org reports against. Treat it with the care of any load-bearing code change.

The exercise enforces:

- **Peer review.** Two people (contributor + reviewer) must approve. The reviewer is either the owning team's engineer or, for cross-team-shared suites, the eval-team lead.
- **CI validation.** Schema, tag catalog, `case_id` uniqueness, `case_hash` uniqueness (a duplicate is a copy-paste error), plane `plan --dry-run` per runner adapter (the runners that will consume this suite can *plan* a run without failing).
- **Cost preview.** The PR comment estimates the *added* judge cost per gate run: `Δ cases × mean-tier-cost × M rubrics`. A PR that adds 500 cases with a per-run cost bump of $20 is a decision the owning team makes deliberately, not a surprise on next month's invoice.
- **Change log.** The PR body names *why* the case was added — the incident id, the customer report, the feature launch, the drift alert. This is what the runbook cites six months later when the case fires and the current on-call has no context.

For eval-set-as-database (surface B), the equivalent is the four-eyes review, the cost preview on the submission form, and a required "why" field.

### Retention, retirement, and the "delete a case" question

Cases do not get deleted. Two mechanisms cover the legitimate desire to remove one:

- **Retirement.** The case is no longer scored. The `retired_at` column is set; the case still exists for historic-run reproducibility. New `eval_set_hash` computations exclude retired cases. This is the right mechanism for "the feature was removed" or "the case is obsolete."
- **Supersession.** A corrected version replaces the old one. The old case is retired with a `superseded_by` pointer to the new case's id. The new case gets a new id and its own history. This is the right mechanism for "the expected output was wrong."

Two edge cases the exercise pins down:

- **PII in a case.** A case captured from real support traffic contains a customer's name. Retire the case; the row remains for reproducibility. If the row's *content* must be scrubbed for a compliance request, do it via a schema-preserving redaction that leaves the case's shape intact — do not `DELETE FROM eval_cases`. A hard delete breaks the reproducibility of every historic run that used the case.
- **A safety-sensitive case.** A jailbreak-attack case that succeeded is stored, but the successful attack text is not — chapter 02 of mod-108 defined the payload-management contract. The case row references an out-of-repo store (a sealed bucket, an encrypted archive) and stores only the hash. Retention of the payload is a separate contract from retention of the case row.

### Golden as the anchor: the "rot-check" migration flow

The golden set is small and careful. Over time, production traffic drifts (mod-107 detects it), and the golden set no longer covers the shape of usage. The recovery flow the chapter recommends:

- **Rot-check runs monthly.** The mod-107 sampler picks a stratified sample of production traces; a spot-check surfaces the ones that would have failed the golden rubric (as scored by the online loop). These become a new `support_bot_rot_check_2026_08` eval set. This set has looser review — one reviewer, not two — because its role is to *surface* drift, not to *anchor* the scorecard.
- **Quarterly, promote the load-bearing rot-check cases to golden.** The eval-team lead and the owning-team lead review the rot-check misses that reproduced at least twice, add them to golden with full care (two reviewers, description, why-added note), and bump the golden minor version. Every downstream gate picks up the new minor at next run.
- **The rot-check bucket is not the golden bucket.** They are separate suites with separate hashes and separate scorecards. Reporting keeps them separate; a mixed number is the "moving denominator" bug that makes drift undetectable.

This flow is what keeps the golden set relevant without letting it grow uncontrollably or drift each week.

### Where the vendors' test-case surfaces fit

The runners each ship a test-case UI. Braintrust has a Dataset surface with a UI; Weave has `weave.Dataset`; Langfuse has datasets; Promptfoo has YAML files first-class; DeepEval has dataset objects; RAGAS has HF-dataset-shaped inputs.

The plane's stance:

- **The schema is the source of truth.** Every case exists in `eval_cases`; the vendors' native dataset objects are *views* the runner adapter reifies at run time.
- **The adapter is responsible for the sync.** When a runner needs the eval set as a Braintrust Dataset, the adapter creates or updates the Dataset from the `eval_cases` rows immediately before the run and tags it with the `eval_set_hash`. The run's outputs are then re-normalised back into `eval_results`.
- **Do not let the vendor's UI become a second write path.** A case added directly in the Braintrust UI that never round-trips into `eval_cases` is a case that no longer participates in the reproducibility contract. Chapter 04 covers the adapter's sync contract; if the vendor's UI is where product teams *want* to work, the plane's job is to make the adapter's sync bidirectional (import the vendor's changes as a proposed contribution requiring the same review as any other).

### What a healthy test-case management surface looks like at year one

Signals that the surface is working:

- A non-engineer product-team member has independently added at least one case in the past quarter (measured directly — check the `contributor` column).
- No case has been added directly to a runner's UI without a corresponding `eval_cases` row within 24 hours (audit query).
- The golden set has grown by 10 – 30 % over the year and the rot-check → golden promotion flow has run at least twice.
- Every case in the runbook has an `owner_team` that the on-call can page.
- The reproducibility check (chapter 02) passes on every historic run whose `eval_set_hash` is still resolvable to a case set.

Signals it is failing:

- The eval set is exactly what it was at kickoff. Product-team contributions are zero.
- A product team maintains a private CSV.
- The scorecard the org reports and the scorecard the team trusts diverge.
- A case fires and no one knows who owns it.
- A "delete this case" request has been fulfilled with a hard delete, breaking a historic run.

## Summary

- The test-case management surface has one job: **let product teams contribute cases safely**, without touching the harness, and keep every case tied to an owner, a version, and an `eval_set_hash`.
- Two contribution surfaces: **eval-set-as-code** (YAML in repo, PR review, CI validation) and **eval-set-as-database** (UI plus API, four-eyes review). Both write to the same schema.
- The **`case_id`** is stable across suite versions, meaningful (not `CASE-000042`), and never reused. The **`case_hash`** changes with the body; the id does not.
- The **`eval_set_hash`** is a SHA-256 over the sorted `[{case_id, case_hash}]` canonical form. Order-independent, content-sensitive, whitespace-insensitive. A run always names a specific hash — never "the latest."
- Three subset roles per suite: **golden** (small, careful, PR-gated), **regression** (grows monotonically, one row per past incident), **rot-check** (monthly, sampled from production, promotes to golden quarterly).
- **Tags** use a controlled vocabulary along multiple axes (feature / capability / criticality / tenant-visible). Every case has an **`owner_team`** for runbook routing.
- **Semver on the suite** — patch = additive, minor = backward-compatible, major = shape change. The hash is the trust anchor; the semver is the human-coordination label.
- **Change review** requires two approvers, CI validation, a cost preview on the PR comment, and a "why" note per case.
- **Retirement, not deletion.** PII / safety-sensitive content is retired (row preserved) — hard delete breaks historic reproducibility.
- The runners' native dataset surfaces are **views** the adapter reifies; the schema is the source of truth.

Chapter 04 opens the runner-adapter contract — the plane's control-plane / data-plane split and the interface every runner speaks so that a `EvalRunSpec` in produces an `EvalResultRow` out, regardless of which runner picked it up.
