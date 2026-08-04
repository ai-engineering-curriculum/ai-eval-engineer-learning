# Why Eval-Gated CI/CD for Prompts, Chains, and Agents

## Motivation

By this point in the track you can define what "good" means for a surface (mod-101), you can emit traces that let you reconstruct a single run (mod-102), you can evaluate a trajectory (mod-103), you can run a calibrated judge (mod-104), and you can compute the RAG triad on real retrieval (mod-105). What you do not yet have is a place in the software lifecycle where those signals are consulted *before* a change reaches a user.

That place is CI/CD. A prompt tweak, a chain re-wiring, a tool-schema change, a retriever swap, or a model upgrade is a code change. Every code change goes through pull-request review, merges to main, and rolls out. The team has weeks of muscle memory around gating that flow on **type checks and unit tests**. Adding evaluation to the same flow — a job that runs on the PR, produces a report a reviewer reads, and blocks the merge on a regression — is what makes evaluation *load-bearing* rather than a dashboard someone looks at once a week.

Without a gate, the eval work of mods 101–105 is theatre. A judge score drops, and nobody notices until an on-call. A retriever change silently poisons faithfulness for a subset of tenants, and the surface degrades between the sprint that touched it and the sprint that reads Grafana. A model upgrade that improves aggregate quality regresses one specific criterion your product cares about, and the PR reviewer had no way to see it. The gate is what closes the loop between "we can measure it" and "we can prevent shipping the regression."

This chapter fixes the vocabulary the rest of the module reuses — **eval gate**, **regression fixture**, **replay set**, **pre-registered threshold**, **release-gate rubric**, **canary**, **progressive rollout** — and states the five properties a serviceable gate must have. Chapters 02–06 build each property in turn.

## Core concepts

### The eval gate: definition and non-definition

An **eval gate** is a stage in the change lifecycle at which an automated evaluation runs, a pass/fail decision is produced against pre-registered thresholds, and the decision blocks progression (merge, canary promotion, full rollout) until either the gate passes or a human overrides it with a documented reason.

Two things it is *not*:

- It is not a dashboard. A dashboard is passive — someone must look at it. A gate is active — the change is blocked until the decision is made. Dashboards support the gate; they do not replace it.
- It is not a unit test. A unit test asserts deterministic behaviour on a hard-coded input. An eval gate scores a *stochastic* system against a rubric on a *replay-representative* set of inputs, and the assertion is a **statistical** threshold, not an equality check. The engineering practices are similar; the failure modes and the interpretation are different.

The industry vocabulary is fragmenting — vendors call this "CI evals," "PR evals," "eval-driven development," "TestOps for LLMs," or "release-gate scoring." Pick one term and use it consistently in your repo; this module uses **eval gate** because the mechanism is a gate and the object being gated is a change.

### The three gate points in a change's life

There are three natural places to run an eval gate for an LLM surface, and a mature team runs at least two of them. Skipping the first is common; skipping the second and third is a mistake.

| Gate | Trigger | Purpose | Blocks |
|---|---|---|---|
| **Pre-commit / local** | `git commit`, `make eval` | Cheap smoke suite — catches obvious breakage before the reviewer sees the PR | Nothing (advisory) or `git commit` if the team accepts the hook |
| **PR / merge gate** | `pull_request` event | Full offline eval suite; produces the report the reviewer reads | Merge to main |
| **Deploy gate** | Release branch → canary → progressive rollout | Runs the same offline suite against the built artefact; canary + shadow-eval against a slice of production traffic | Canary promotion; then progressive-rollout stages |

Chapter 05 wires the **PR gate**; chapter 06 wires the **deploy gate**; the pre-commit hook is a stretch goal in exercise-01. The three gates re-use the same eval config (chapter 04) — the difference is the runner, the budget, and the blocking severity, not the eval logic.

### The five properties of a serviceable gate

A gate that ships to a team of engineers who did not build it has to satisfy five properties. Each maps to a chapter of this module.

1. **Reproducible.** Two runs of the gate on the same commit produce the same pass/fail decision. That requires pinning the model version, the sampling seed, the retrieval index hash, the judge model, and the rubric prompt hash. Chapter 02 covers this.
2. **Pre-registered.** The pass/fail thresholds are written down and versioned *before* the change lands. A threshold negotiated after seeing the number is not a threshold; it is a rationalisation. Chapter 03 covers this.
3. **Declarative.** The eval config is a file — YAML, pytest fixture, a JSON manifest — that lives in the repo, is code-reviewable, and is portable across runners. A gate whose config is "click this in the vendor UI" is not a gate; it is a manual step in a hoodie. Chapter 04 covers this.
4. **Runnable from CI.** The gate runs on the PR CI runner (usually GitHub Actions, sometimes GitLab / Buildkite / CircleCI) and posts a machine-readable report the reviewer can read without leaving the PR. Chapter 05 covers this.
5. **Wired into deploy.** The gate result is an *input* to the deploy pipeline the platform team owns — the canary is not promoted, the progressive rollout does not advance, if the gate is failing. Chapter 06 covers the interface.

A gate that has properties 1–4 but not 5 catches regressions at PR time and misses everything downstream of merge (drift, upstream data change, latent bug that only surfaces on real traffic). A gate that has property 5 without 1–4 promotes noisy canaries and rolls back for reasons nobody can defend. All five matter.

### The regression fixture and the replay set

The unit of work an eval gate scores is a **fixture** — one input, plus the ground truth or reference material the rubric compares against, plus any pinned state the run depends on (model version, retrieval index snapshot, tool stubs). A collection of fixtures held together is a **replay set**. Chapter 02 defines the shape and lifecycle in detail; two vocabulary rules to fix now:

- A fixture is derived from **one real production trace** (or a hand-authored one that mimics a real trace shape). It is not a synthetic prompt somebody made up in a design meeting. Fixtures pulled from real traces are how the gate stays anchored to real product bugs; a synthetic fixture pulled from a benchmark is how gates start reporting numbers that do not move on real regressions.
- The replay set is **held separately from the calibration gold set** (mod-104 chapter 04). Using calibration data as your CI fixtures means every threshold miss re-opens a calibration debate; you never accumulate signal about drift.

### Pre-registered thresholds and pass/fail semantics

The gate's job is to answer a binary question: *does this change ship?* Turning a set of continuous eval scores into a binary decision requires a rule, and that rule has to be written down *before* the change is proposed. Chapter 03 covers the mechanics. Three shapes to know:

- **Absolute threshold.** "Faithfulness must be ≥ 0.85 on the RAG replay set." Simple; the risk is that scores that drift down slowly never trip until they cross the wire.
- **Delta threshold.** "Faithfulness must not drop by more than 0.02 vs main." Catches drift better; requires a stable baseline against which to compute the delta.
- **Composite threshold.** "Faithfulness ≥ 0.85 AND groundedness delta ≥ –0.02 AND cost per query delta ≤ +5%." Necessary for surfaces with cost/latency SLOs alongside quality. Every clause must have a stated rationale and a stated owner.

### Regressions are first-class blockers

The behaviour of the gate on a regression is what determines whether the gate actually functions. Two anti-patterns to avoid:

- **Warn-only forever.** Teams often ship the gate in "advisory mode" for a week, then never flip the switch. The gate then produces a report nobody reads and blocks nothing. Ship the switch flip as a scheduled follow-up in the same PR that adds the gate, or you never get to blocking mode.
- **Any engineer can auto-override.** If merging past a failing gate is a click, everyone does it under deadline pressure. Overrides must (a) require a documented rationale on the PR, (b) auto-file a follow-up issue owned by a named person, and (c) show up on a dashboard the on-call reviews. Chapter 03 walks the runbook shape.

The default posture: **a failing gate blocks merge; overrides are rare and audited**. This is the same posture the team already has for a failing type check; the eval gate is a first-class version of that.

### The eval-gated pipeline end-to-end

A single mental picture for the whole module:

```
developer opens PR
  │
  ▼
[PR gate — chapter 05]
  │  runs eval config (chapter 04) against replay set (chapter 02)
  │  scored against thresholds (chapter 03)
  │  posts machine-readable report as PR comment / check
  │
  ▼ (pass)
merge to main
  │
  ▼
build release artefact
  │
  ▼
[deploy gate — chapter 06]
  │  runs same eval config against artefact
  │  canary against slice of production traffic (offline-scored + shadow-eval)
  │  progressive rollout advances iff gate stays green
  │
  ▼
full rollout
  │  online monitor (mod-107) picks up here
```

Every arrow is a mechanised step. Every gate reads the same config file. Every threshold is versioned in the repo. Every regression alert links to a runbook naming an owner. That is the shape the rest of this module builds.

### What this module does *not* cover

- **Statistical methodology for the thresholds themselves.** Choosing the specific numeric threshold and the confidence interval around it is model-evaluation-engineer peer territory (Model-Development family, level 30). This module reuses the mod-104 calibration outputs and the mod-105 RAG-triad numbers; it does not re-derive them.
- **The full online-eval loop against production traffic.** That is mod-107. Chapter 06 covers the *interface* between the deploy gate and the online loop, not the loop itself.
- **Deploy tooling.** The deploy pipeline is owned by the platform team (Argo, Spinnaker, GitHub Actions deploys, Kubernetes rollouts, service-mesh traffic-splitting). This module is the eval side of the contract; chapter 06 states the interface, not the implementation.
- **Safety and guardrail-specific gates.** Guardrail measurement has its own module (mod-108) and its own release-gate posture (blocking, no soft-fail). This module's gate framework covers safety gates; mod-108 defines what they measure.

## Summary

- An **eval gate** blocks a change (merge, canary promotion, rollout stage) on a pass/fail decision against pre-registered thresholds. It is not a dashboard and is not a unit test.
- Three gate points: pre-commit (advisory), PR / merge (blocks merge), deploy (blocks promotion). A mature team runs at least the PR and deploy gates.
- Five properties of a serviceable gate: **reproducible** (ch. 02), **pre-registered** (ch. 03), **declarative** (ch. 04), **runnable from CI** (ch. 05), **wired into deploy** (ch. 06).
- Fixtures come from real production traces, not benchmark prompts. The replay set is held separately from mod-104's calibration gold set.
- Pre-registered thresholds come in three shapes (absolute / delta / composite). Regressions are first-class blockers; overrides are rare, documented, and audited.
- This module is the CI/CD side of the eval program. It reads the mod-102 trace shape, the mod-104 judge, and the mod-105 RAG triad; it feeds the mod-107 online loop and the mod-112 release-gate architecture.

Chapter 02 walks the reproducibility property: snapshotting production traces into a replay set with pinned model, seed, and retrieval-index hashes.
