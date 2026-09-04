# Resources for mod-107-online-eval-and-regression (Online Evaluation and Regression Detection at the App Layer)

Primary references cited across the chapters and exercises. Prefer the official docs and the original papers over blog posts — the trace-backend, drift-detection, and A/B-platform surfaces move on a monthly cadence and the summaries drift. Read from the URL each time you plan to build against a specific behaviour.

## Trace backends and online-scoring runtimes

- **Arize Phoenix — documentation.** <https://docs.arize.com/phoenix>. Trace backend used as the loop's substrate; span annotations, online evaluators, and dashboards referenced from chapters 02 and 07.
- **Phoenix — online evaluations guide.** <https://docs.arize.com/phoenix/evaluation/how-to-evals/online-evaluations>. The runtime-scoring surface for chapter 02's judge runner.
- **Langfuse — documentation.** <https://langfuse.com/docs>. Traces, scores, and evaluators; Langfuse's `Score` object is chapter 02's per-metric row in Langfuse-backed loops.
- **Langfuse — model-based evaluation.** <https://langfuse.com/docs/scores/model-based-evals>. Runtime evaluator wiring referenced from chapter 02.
- **W&B Weave — documentation.** <https://weave-docs.wandb.ai/>. `weave.Evaluation`, `weave.op`, scorers attached to traces; chapter 02's third vendor SDK.
- **W&B Weave — online evaluations.** <https://weave-docs.wandb.ai/guides/evaluation/scorers>. Scorer definitions used as runtime judges.
- **Braintrust — documentation.** <https://www.braintrust.dev/docs>. Datasets, scorers, experiments; chapter 02's fourth vendor SDK.
- **Braintrust — online evaluation.** <https://www.braintrust.dev/docs/guides/online-evals>. `online.score` runtime hook and comparison-first dashboards.
- **LangSmith — production observability.** <https://docs.smith.langchain.com/observability>. LangChain-native trace backend with online-evaluator primitives.
- **OpenTelemetry — probabilistic samplers.** <https://opentelemetry.io/docs/specs/otel/trace/sdk/#probability-samplers>. `TraceIdRatioBased` and parent-based samplers; the deterministic-hash sampler in chapter 02 is this shape.
- **OpenInference — semantic conventions.** <https://github.com/Arize-ai/openinference/tree/main/spec>. LLM span attribute conventions the scored-row schema in chapter 02 aligns to.

## Drift detection libraries

- **Alibi Detect — documentation.** <https://docs.seldon.io/projects/alibi-detect/en/stable/>. MMD, classifier-based two-sample tests, KS, chi-squared, embedding-drift detectors; primary reference for chapter 03 and exercise-02.
- **Alibi Detect — repo.** <https://github.com/SeldonIO/alibi-detect>. OSS implementations; useful when reading the exact test-statistic derivation.
- **NannyML — documentation.** <https://nannyml.readthedocs.io/>. CBPE-style performance estimation, univariate and multivariate drift, DLE. Chapter 03's second reference.
- **NannyML — repo.** <https://github.com/NannyML/nannyml>. OSS source.
- **Evidently AI — documentation.** <https://docs.evidentlyai.com/>. Batteries-included drift-report shapes; the fastest way from a scored-row DataFrame to a dashboard.
- **Evidently AI — repo.** <https://github.com/evidentlyai/evidently>. OSS source.
- **River — online change-point detection.** <https://riverml.xyz/latest/api/drift/>. Streaming CUSUM, Page-Hinkley, ADWIN implementations; useful when chapter 03's CUSUM row of the decision matrix needs a Python implementation.

## Drift statistics — primary sources

- **Population Stability Index — origins.** Karakoulas, G. J. "Empirical Validation of Retail Credit-Scoring Models." (Wharton, 2004; reproduced in retail-credit-scoring literature). PSI is folklore in credit scoring rather than a single-paper introduction; the < 0.1 / 0.1–0.25 / > 0.25 bands in chapter 03 come from this practitioner tradition. Cross-reference: <https://scikit-lego.readthedocs.io/en/latest/> and the Alibi Detect PSI notebook.
- **Kolmogorov–Smirnov two-sample test.** Massey, F. J. "The Kolmogorov-Smirnov Test for Goodness of Fit." *Journal of the American Statistical Association*, 1951. Standard reference; every stats textbook covers it.
- **Cramér–von Mises two-sample test.** Anderson, T. W. "On the Distribution of the Two-Sample Cramér-von Mises Criterion." *Annals of Mathematical Statistics*, 1962.
- **Maximum Mean Discrepancy.** Gretton, A., Borgwardt, K., Rasch, M., Schölkopf, B., Smola, A. "A Kernel Two-Sample Test." *JMLR* 13, 2012. <https://jmlr.org/papers/v13/gretton12a.html>. The reference for MMD-based two-sample testing used in chapter 03's embedding-drift row.
- **Classifier-based two-sample tests.** Lopez-Paz, D., Oquab, M. "Revisiting Classifier Two-Sample Tests." *ICLR* 2017. <https://arxiv.org/abs/1610.06545>. The "train a binary classifier, use its AUC as the statistic" pattern cited in chapter 03.
- **CUSUM.** Page, E. S. "Continuous Inspection Schemes." *Biometrika* 41, 1954. Original derivation.
- **Segment discovery — SliceFinder.** Chung, Y., Kraska, T., Polyzotis, N., Tae, K. H., Whang, S. E. "Automated Data Slicing for Model Validation." *ICDE* 2019. <https://arxiv.org/abs/1807.06068>. The default segment-discovery tool referenced in chapter 03 and exercise-02.
- **Segment discovery — DivExplorer.** Pastor, E., de Alfaro, L., Baralis, E. "Looking for Trouble: Analyzing Classifier Behavior via Pattern Divergence." *SIGMOD* 2021. <https://dl.acm.org/doi/10.1145/3448016.3457284>. Alternate segment-discovery reference for the automatic-cohort-surfacing stretch goal.
- **Multiple testing — Benjamini–Hochberg.** Benjamini, Y., Hochberg, Y. "Controlling the False Discovery Rate: A Practical and Powerful Approach to Multiple Testing." *JRSS-B* 57, 1995. The FDR-controlling procedure applied in chapter 03 and exercise-02.

## Sequential testing and anytime-valid inference

- **Optimizely Stats Engine — always-valid inference.** Johari, R., Pekelis, L., Walsh, D. J. "Always Valid Inference: Continuous Monitoring of A/B Tests." *arXiv:1512.04922*, 2015. <https://arxiv.org/abs/1512.04922>. The paper behind Optimizely's stats engine; the peeking-inflation demonstration in chapter 05 anchors on the empirical numbers here.
- **Optimizely Stats Engine — vendor writeup.** <https://www.optimizely.com/optimization-glossary/statistical-significance/>. The product-side companion.
- **mSPRT — theory.** Wald, A. *Sequential Analysis.* Wiley, 1947. Foundational; the mSPRT construction extends this to a mixture prior.
- **Deng, Lu, Chen — mSPRT for online experimentation.** Deng, A., Lu, J., Chen, S. "Continuous Monitoring of A/B Tests without Pandora's Box." *IEEE DSAA* 2016. <https://ieeexplore.ieee.org/document/7796923>. The A/B-platform-oriented treatment of mSPRT referenced in chapter 05.
- **Confidence sequences — foundational paper.** Howard, S. R., Ramdas, A., McAuliffe, J., Sekhon, J. "Time-uniform, nonparametric, nonasymptotic confidence sequences." *Annals of Statistics* 49(2), 2021. <https://arxiv.org/abs/1810.08240>. The reference construction for the empirical-Bernstein and Hoeffding CSs in chapter 05 and exercise-04.
- **Empirical-Bernstein bounds.** Maurer, A., Pontil, M. "Empirical Bernstein Bounds and Sample Variance Penalization." *COLT* 2009. <https://arxiv.org/abs/0907.3740>. The variance-adaptive bound used in chapter 05's default recommendation for bounded rubric scores.
- **Betting-style confidence sequences.** Waudby-Smith, I., Ramdas, A. "Estimating Means of Bounded Random Variables by Betting." *JRSS-B*, 2024. <https://arxiv.org/abs/2010.09686>. The betting-based construction cited in chapter 05's stretch alternatives.
- **Deng-Xu-Kohavi — pitfalls of online controlled experiments.** Kohavi, R., Deng, A., Xu, Y. "Trustworthy Online Controlled Experiments: Five Puzzling Outcomes Explained." *KDD* 2012. <https://ai.stanford.edu/~ronnyk/2012PuzzlingOutcomesInControlledExperiments.pdf>. Peeking, novelty effects, and other patterns chapter 05 refers to.
- **Kohavi, Tang, Xu — book.** *Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing.* Cambridge University Press, 2020. The single most cited practitioner reference for the A/B-testing side of chapter 06; the pre-registration contract and guardrail metrics vocabulary derive from this treatment.
- **Statsig — sequential testing docs.** <https://docs.statsig.com/experiments-plus/sequential-testing>. Vendor implementation of mSPRT; chapter 05 references the Statsig-published empirical numbers.
- **Sequential (R package) — Kulldorff & Silva.** <https://cran.r-project.org/package=Sequential>. Sequential MaxSPRT / CMaxSPRT for rates and counts; chapter 05's rates-and-counts reference implementation.
- **`confseq` (Python) — Howard & Ramdas reference implementation.** <https://github.com/gostevehoward/confseq>. The reference implementation for the confidence-sequence boundaries used in exercise-04.

## CUPED, variance reduction, and sanity checks

- **CUPED — original paper.** Deng, A., Xu, Y., Kohavi, R., Walker, T. "Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data." *WSDM* 2013. <https://exp-platform.com/Documents/2013-02-CUPED-ImprovingSensitivityOfControlledExperiments.pdf>. The variance-reduction technique defined in chapter 06 and delivered as covariate in exercise-05.
- **CUPED — Microsoft ExP writeup.** <https://exp-platform.com/blog/>. Practitioner posts on CUPED implementation and interaction with metric definitions.
- **Sample ratio mismatch — Fabijan et al.** Fabijan, A., Gupchup, J., Gustafson, S., Omhover, J.-F., Qin, W., Vermeer, L., Dmitriev, P. "Diagnosing Sample Ratio Mismatch in Online Controlled Experiments." *KDD* 2019. <https://www.exp-platform.com/Documents/2019_KDD_DiagnosingSampleRatioMismatchInOnlineControlledExperiments.pdf>. The SRM check referenced in chapter 06 and exercise-05.
- **A/B testing — Kohavi practitioner materials.** <https://exp-platform.com/>. Microsoft Experimentation Platform's archive of publications; the umbrella source for CUPED, guardrails, SRM, novelty effects, and interaction detection.
- **Guardrail metrics — Netflix practitioner writeup.** Xie, H., Aurisset, J. "Improving the Sensitivity of Online Controlled Experiments: Case Studies at Netflix." *KDD* 2016. <https://netflixtechblog.com/its-all-a-bout-testing-the-netflix-experimentation-platform-4e1ca458c15>. Guardrail-metric shape referenced in chapter 06.

## A/B experimentation platforms

- **Statsig — documentation.** <https://docs.statsig.com/>. Metric definitions, experiments, sequential testing, holdouts; chapter 06's default platform reference.
- **Eppo — documentation.** <https://docs.geteppo.com/>. Warehouse-native experimentation; CUPED and pre-registration surface.
- **Optimizely Experimentation — documentation.** <https://docs.developers.optimizely.com/web-experimentation/>. Alternate vendor reference; stats engine described above.
- **LaunchDarkly Experimentation — documentation.** <https://docs.launchdarkly.com/home/experimentation>. Flag-driven experimentation; ties directly to the feature-flag rollback shape in chapter 04.
- **Split — documentation.** <https://help.split.io/hc/en-us/categories/360003245692-Experimentation>. Fifth vendor reference.
- **GrowthBook — open-source alternative.** <https://docs.growthbook.io/>. OSS A/B platform; useful when the org does not yet have a vendor.

## Canary, shadow, and progressive rollout

- **Argo Rollouts — documentation.** <https://argoproj.github.io/argo-rollouts/>. Canary, blue/green, and progressive-rollout primitives on Kubernetes; `AnalysisTemplate` is the metric-gate surface chapter 04's script targets.
- **Argo Rollouts — analysis and metrics.** <https://argoproj.github.io/argo-rollouts/features/analysis/>. The `AnalysisTemplate` reference for the exercise-03 stretch goal.
- **Flagger — documentation.** <https://flagger.app/>. GitOps-based progressive-delivery operator; canary analysis with metrics gates.
- **Flagger — canary custom resource.** <https://docs.flagger.app/usage/how-it-works>. The `Canary` and `MetricTemplate` shapes for the exercise-03 stretch goal.
- **Spinnaker — canary analysis.** <https://spinnaker.io/docs/guides/user/canary/>. Long-standing canary tooling; Kayenta is the analysis engine.
- **Kubernetes — Deployment strategies.** <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy>. Rolling-update and recreate strategies underlying most canary systems.
- **OpenFeature — specification.** <https://openfeature.dev/specification/>. Vendor-neutral feature-flag spec; chapter 04's flag-flip rollback interfaces against this.
- **LaunchDarkly — progressive delivery.** <https://docs.launchdarkly.com/home/experimentation/progressive-delivery>. Vendor implementation of the flag-driven canary and rollback pattern.
- **Google SRE Book — chapter 27, Reliable Product Launches at Scale.** <https://sre.google/sre-book/reliable-product-launches/>. Canary and staged-rollout mental model referenced in chapter 04.
- **Google SRE Workbook — canarying releases.** <https://sre.google/workbook/canarying-releases/>. The canonical treatment of canary analysis and rollback criteria.

## Dashboarding and alerting surfaces

- **Grafana — documentation.** <https://grafana.com/docs/grafana/latest/>. General-purpose dashboard reference for chapter 07's on-call view.
- **Grafana — alerting.** <https://grafana.com/docs/grafana/latest/alerting/>. Alert routing and severity used in exercise-06.
- **Apache Superset — documentation.** <https://superset.apache.org/docs/intro>. Second general-purpose dashboard option for chapter 07.
- **Metabase — documentation.** <https://www.metabase.com/docs/latest/>. Third dashboard option; commonly present in product-analytics stacks.
- **Looker — documentation.** <https://cloud.google.com/looker/docs>. Fourth option; the LookML modelling layer maps well to cohort-key discipline from chapter 04.
- **PagerDuty — alerting best practices.** <https://developer.pagerduty.com/docs/response-plays>. Response-play and severity-routing patterns for the on-call channel in chapter 07 and exercise-06.

## Model API pinning (cross-reference to mod-106; needed for judge-drift attribution)

- **OpenAI — model versioning.** <https://platform.openai.com/docs/models>. Snapshot suffix (`gpt-4o-<YYYY-MM-DD>`) and alias-vs-snapshot distinction referenced in chapter 03's judge-drift monitor.
- **Anthropic — model overview and versioning.** <https://docs.anthropic.com/en/docs/about-claude/models>. Snapshot naming (`claude-3-5-sonnet-<YYYYMMDD>`) referenced in chapter 03's judge-drift monitor.
- **AWS Bedrock — model versioning.** <https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html>. Snapshot pinning in the model ARN.

## Governance and program-owner cross-references (deeper coverage in mod-112)

- **NIST AI Risk Management Framework (AI 100-1, 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. The Measure and Manage functions cover the online loop's alerting posture; drift monitors and cohort gates are Measure artefacts, rollback contracts are Manage artefacts.
- **NIST AI RMF Generative AI Profile (AI 600-1, 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Continuous-monitoring suggested actions map to this module's drift monitors and confidence sequences.
- **ISO/IEC 42001:2023 — AI management systems.** <https://www.iso.org/standard/81230.html>. Continuous-monitoring and change-control controls; the online-loop artefacts are 42001-relevant. Paywalled.
- **EU Artificial Intelligence Act (Regulation (EU) 2024/1689).** <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. Post-market monitoring obligations for high-risk systems (Articles 72 – 73); the online-eval loop is one of the artefacts a high-risk deployment would produce.

## Peer curriculum interfaces

- **`ai-evaluation-engineer-learning` (Governance family, level 25)** — the sign-off owner for release-gate and continuous-monitoring architecture; mod-112 formalises the hand-off from this module.
- **`model-evaluation-engineer-learning` (Model-Development family, level 30)** — the peer that owns full experimental methodology (MDE / power / novelty / network interference); chapter 06 hands off the deep A/B-methodology work to this peer.
- **`senior-ml-engineer-learning` (Model-Development family)** — the product-team peer that co-owns ship / iterate / kill decisions with the A/B platform; chapter 06 names the delegation boundary.
- **`agentic-ai-engineer-learning`** — the agent implementations whose live traffic feeds the online loop; mod-102 chapter references apply.
- **`llm-application-developer-learning`** — the LLM app engineering whose prompt / chain changes flow through the canary gate in chapter 04.

<!-- needs-research: on the next research cycle, re-verify current Statsig / Eppo / Optimizely sequential-testing and CUPED documentation URLs, re-check the Phoenix / Langfuse / Weave / Braintrust online-scoring surfaces (which change on a monthly cadence), and refresh the Argo Rollouts / Flagger `AnalysisTemplate` / `MetricTemplate` references. Verify the current `confseq` Python package location and the Alibi Detect / NannyML documentation entry points. -->
