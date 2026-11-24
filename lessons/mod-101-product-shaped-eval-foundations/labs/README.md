# Labs for mod-101-product-shaped-eval-foundations (Product-Shaped Evaluation Foundations: From Spec to Eval Plan)

The labs turn the artefacts the exercises produced into running code: an offline fixture harness with a merge-gate preflight check, and a simulated sampled-trace online loop. They are complementary — together they are the two halves of the chapter-03 placement decision made concrete.

- [`lab-01-offline-fixture-harness.md`](lab-01-offline-fixture-harness.md) — stand up a minimal Python harness that loads a fixture file, runs a stub surface, applies rule-based scorers, compares SLIs against pre-registered SLOs from a `plan.yaml`, and exits non-zero when a merge-gate SLO fails. The preflight CI check `mod-106` later extends. 60–90 min.
- [`lab-02-online-sampled-trace-loop.md`](lab-02-online-sampled-trace-loop.md) — simulate the online loop on a synthetic trace stream: stratified sampling with a per-cohort cap, a deterministic mock judge, a watch-only SLO, and a page threshold. Rerun against a drifted trace file to watch the SLI move. 45–60 min.
