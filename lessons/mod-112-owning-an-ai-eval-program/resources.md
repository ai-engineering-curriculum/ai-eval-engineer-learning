# Resources for mod-112-owning-an-ai-eval-program (Owning an AI Evaluation Program Across Product, Infra, Safety, and Governance)

Primary references cited across the chapters and exercises. Prefer official standards, primary documentation, and peer-reviewed research over vendor blog posts — the regulatory texts, the frameworks, and the vendor landscapes all evolve, and summaries drift fast. For items marked `needs-research`, re-verify against the source on the next research cycle.

## Program ownership, release engineering, and SRE foundations

Chapter 02's release-gate architecture builds on the long tradition of SRE-style release engineering applied to the eval surface — gates as contracts, thresholds with provenance, rollback as a first-class concern.

- **Beyer, B., Jones, C., Petoff, J., Murphy, N. R. "Site Reliability Engineering: How Google Runs Production Systems." O'Reilly, 2016.** <https://sre.google/sre-book/table-of-contents/>. The reference text; chapter 2 (eliminating toil) and the release-engineering chapters are the discipline chapter 02's gate architecture extends.
- **Beyer, B., Murphy, N. R., Rensin, D. K., Kawahara, K., Thorne, S. "The Site Reliability Workbook." O'Reilly, 2018.** <https://sre.google/workbook/table-of-contents/>. Practical SLO authorship + alerting shape; chapter 02's rollback triggers extend the alerting-on-SLOs chapter.
- **Google SRE Workbook — Implementing SLOs.** <https://sre.google/workbook/implementing-slos/>. The SLI / SLO formalism the gate's thresholds and rollback triggers inherit.
- **Google SRE Workbook — Alerting on SLOs.** <https://sre.google/workbook/alerting-on-slos/>. Burn-rate alert thresholds; chapter 02's rollback-trigger shape.
- **Humble, J., Farley, D. "Continuous Delivery: Reliable Software Releases through Build, Test, and Deployment Automation." Addison-Wesley, 2010.** The deployment-pipeline shape — commit → build → test → deploy — that chapter 02's four-gate topology (offline / canary / ramp / post-ship) sits on top of.
- **Humble, J. — "Blue/Green Deployments" and "Canary Releases."** <https://continuousdelivery.com/implementing/patterns/>. Pattern references for the canary + ramp gates.
- **OpenSLO — specification.** <https://openslo.com/docs/overview>. OSS spec for declaring SLOs as code; exercise 01's `rollback_criteria.yaml` can follow this shape.
- **Prometheus — Alerting rules.** <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>. The common alerting substrate the rollback triggers wire against.
- **Gil Tene — "How NOT to Measure Latency." Strange Loop, 2015.** <https://www.youtube.com/watch?v=lJ8ydIuPFeU>. The reference talk on coordinated omission and percentile reporting; mod-111 ch. 05 depth that chapter 02's threshold-provenance discipline inherits.

## Delegation, team interfaces, and inter-team contracts

Chapter 03's delegation contracts formalise cross-team interfaces as versioned agreements — the pattern Team Topologies calls "Team API" and that distributed-systems contract testing calls a "consumer contract."

- **Skelton, M., Pais, M. "Team Topologies: Organizing Business and Technology Teams for Fast Flow." IT Revolution Press, 2019.** <https://teamtopologies.com/book>. The Team API concept and the four fundamental team types (stream-aligned, enabling, complicated-subsystem, platform) are the vocabulary chapter 03's six peer contracts sit under.
- **Pais, M., Skelton, M. — "Team Interaction Modes."** <https://teamtopologies.com/key-concepts>. Collaboration / X-as-a-Service / Facilitating — chapter 03's peer contracts default to X-as-a-Service with explicit escalation.
- **Fowler, M. — "Consumer-Driven Contracts."** <https://martinfowler.com/articles/consumerDrivenContracts.html>. The distributed-systems analogue to chapter 03's contracts; the "symmetric production" discipline derives from this.
- **Pact — Contract testing.** <https://docs.pact.io/>. The reference contract-testing framework; useful mental model for chapter 03's artefact-and-schema binding.
- **Kim, G., Humble, J., Debois, P., Willis, J., Forsgren, N. "The DevOps Handbook: How to Create World-Class Agility, Reliability, and Security in Technology Organizations." IT Revolution, 2nd ed., 2021.** The cross-team flow discipline chapter 03's quarterly-review cadence sits inside.
- **ThoughtWorks — "You build it, you run it" and shared responsibility patterns.** <https://www.thoughtworks.com/insights/blog/microservices-nothing-new>. Context for why chapter 03's contracts are symmetric rather than one-way.

## Build vs. buy, platform engineering, and eval-platform landscape

Chapter 04 decomposes the eval stack into ten components with build / buy / hybrid decisions. The frame draws on platform-engineering practice and the current app-eval tool landscape.

- **Fournier, C. "The Manager's Path." O'Reilly, 2017.** Chapter on tech-strategy decisions; the build-vs-buy frame is the chapter 04 shape at a people-management level.
- **Winters, T., Manshreck, T., Wright, H. "Software Engineering at Google: Lessons Learned from Programming Over Time." O'Reilly, 2020.** The "it's programming integrated over time" framing is the chapter 04 cadence discipline.
- **CNCF — Platform Engineering Maturity Model.** <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>. Vocabulary and maturity staging for platform-engineering decisions; useful cross-reference for chapter 04's refresh cadence.
- **Fowler, M. — "StranglerFigApplication."** <https://martinfowler.com/bliki/StranglerFigApplication.html>. The migration pattern chapter 04's "trigger-driven out-of-cadence review" assumes when a component swap is needed.

**App-eval platform options** (chapter 04's ten-component matrix references these; verify current status):

- **OpenTelemetry — specification + GenAI semantic conventions.** <https://opentelemetry.io/docs/specs/otel/> and <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. The vendor-neutral substrate for trace collection and span attributes; the shape most of the tools below converge on.
- **OpenInference — semantic conventions.** <https://github.com/Arize-ai/openinference/tree/main/spec>. Alternate GenAI span conventions widely used by vendor runtimes.
- **Arize Phoenix — documentation.** <https://docs.arize.com/phoenix>. OSS trace backend + evaluator surface.
- **Langfuse — documentation.** <https://langfuse.com/docs>. OSS trace + evaluator platform (also hosted); EU-region SKU.
- **Braintrust — documentation.** <https://www.braintrust.dev/docs>. Hosted eval + prompt-management platform.
- **Weights & Biases Weave — documentation.** <https://weave-docs.wandb.ai/>. W&B's LLM evaluation surface.
- **LangSmith — documentation.** <https://docs.smith.langchain.com/>. LangChain-native backend.
- **Promptfoo — documentation.** <https://www.promptfoo.dev/docs/intro/>. OSS CLI + CI integration for prompt / eval runs.
- **DeepEval — documentation.** <https://docs.confident-ai.com/>. OSS evaluation framework (Confident AI).
- **RAGAS — documentation.** <https://docs.ragas.io/>. OSS evaluation for RAG pipelines.
- **Humanloop — documentation.** <https://humanloop.com/docs>. Hosted eval + prompt management.
- **Galileo — documentation.** <https://docs.galileo.ai/>. Hosted LLM observability + eval.
- **Patronus AI — documentation.** <https://docs.patronus.ai/>. Hosted eval + guardrails.
- **Argilla — documentation.** <https://docs.argilla.io/>. OSS human annotation + dataset management.
- **Label Studio — documentation.** <https://labelstud.io/guide/>. OSS annotation UI.
- **Helicone — documentation.** <https://docs.helicone.ai/>. OSS + hosted LLM observability + cost accounting.
- **Datadog LLM Observability.** <https://docs.datadoghq.com/llm_observability/>. Datadog's LLM observability surface for teams already standardised on Datadog.
- **New Relic AI Monitoring.** <https://docs.newrelic.com/docs/ai-monitoring/intro-to-ai-monitoring/>. New Relic's equivalent.

**Cloud-native provider eval (chapter 04 component 10):**

- **Google Vertex AI — Gen AI evaluation service.** <https://cloud.google.com/vertex-ai/generative-ai/docs/models/evaluation-overview>. Vertex-managed eval service.
- **AWS Bedrock — Evaluations.** <https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html>. Bedrock-managed model and knowledge-base evaluations.
- **Azure AI Foundry — evaluation.** <https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/evaluation-approach-gen-ai>. Azure's managed eval surface.

**Safety / guardrail infrastructure (chapter 04 component 7):**

- **Llama Guard — model card and documentation.** <https://ai.meta.com/research/publications/llama-guard-llm-based-input-output-safeguard-for-human-ai-conversations/>. Meta's open safety classifier.
- **NVIDIA NeMo Guardrails — documentation.** <https://docs.nvidia.com/nemo/guardrails/latest/>. Programmable guardrail framework (Colang DSL).
- **Guardrails AI — documentation.** <https://www.guardrailsai.com/docs>. OSS validation framework.
- **Microsoft Presidio — documentation.** <https://microsoft.github.io/presidio/>. PII detection and anonymisation.
- **OpenAI Moderation API.** <https://platform.openai.com/docs/guides/moderation>. OpenAI's content classifier.
- **Azure AI Content Safety.** <https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview>. Microsoft's hosted content classifier.
- **garak — LLM vulnerability scanner.** <https://github.com/NVIDIA/garak>. Red-team toolkit widely used for jailbreak and prompt-injection coverage.
- **PyRIT (Python Risk Identification Tool) — Microsoft.** <https://github.com/Azure/PyRIT>. Automated risk identification for generative AI.

<!-- needs-research: on the next research cycle, re-verify the eval-platform / safety-tool landscape — vendor renames, deprecations, and new SKUs re-shuffle roughly quarterly; refresh URLs and confirm the components each vendor still covers today. -->

## Incident response, retrospectives, and post-mortem culture

Chapter 05's investment loop sits on top of the SRE / incident-response literature — blameless post-mortems, root-cause taxonomies, and action-item accountability.

- **Google SRE Book — Postmortem Culture: Learning from Failure (chapter 15).** <https://sre.google/sre-book/postmortem-culture/>. The reference blameless-postmortem shape; chapter 05's retrospective enqueue moment extends it.
- **Allspaw, J. — "Blameless Postmortems and a Just Culture." Etsy, 2012.** <https://www.etsy.com/codeascraft/blameless-postmortems/>. The canonical write-up on blameless retros.
- **Rother, M. "Toyota Kata: Managing People for Improvement, Adaptiveness and Superior Results." McGraw-Hill, 2010.** The improvement-kata and PDCA cadence; chapter 05's monthly ledger review is a PDCA ceremony.
- **Google SRE — Managing Incidents.** <https://sre.google/sre-book/managing-incidents/>. Incident commander, communications lead, operations lead; chapter 05's `incident_commander` field sits inside this frame.
- **Forsgren, N., Humble, J., Kim, G. "Accelerate: The Science of Lean Software and DevOps." IT Revolution, 2018.** The MTTR / change-fail-rate telemetry; chapter 05's investment ledger produces equivalents for the eval surface.
- **Learning From Incidents in Software — community site.** <https://www.learningfromincidents.io/>. Practitioner essays on incident learning that go deeper than SRE-book length.

## OWASP, MITRE ATLAS, and attack-pattern taxonomies

Chapter 05 § classification and the fixtures reference published attack / failure taxonomies.

- **OWASP Top 10 for Large Language Model Applications.** <https://genai.owasp.org/llm-top-10/>. The reference catalogue of LLM-specific vulnerabilities; chapter 02's offline-gate safety threshold references this.
- **MITRE ATLAS — Adversarial Threat Landscape for Artificial-Intelligence Systems.** <https://atlas.mitre.org/>. The ATT&CK-style framework for AI-system adversary tactics and techniques; chapter 05's attack-signal enrichment maps findings to ATLAS technique ids.
- **NIST AI 100-2 E2023 — "Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations."** <https://doi.org/10.6028/NIST.AI.100-2e2023>. Vocabulary for the attack taxonomy; useful cross-reference when authoring the chapter 05 taxonomy per family.
- **AI Incident Database — Partnership on AI.** <https://incidentdatabase.ai/>. Public repository of AI incidents; useful source for exercise 04's "realistic hypothetical" when no in-scope incident is available.

## Model cards, system cards, and transparency reporting

Chapter 06's card slice sits on the model-card / system-card lineage.

- **Mitchell, M., Wu, S., Zaldivar, A., Barnes, P., Vasserman, L., et al. "Model Cards for Model Reporting." *FAT\* 2019*.** <https://arxiv.org/abs/1810.03993>. The reference model-card paper; the shape chapter 06's slice inherits and extends.
- **Gebru, T., Morgenstern, J., Vecchione, B., Vaughan, J. W., Wallach, H., et al. "Datasheets for Datasets." *CACM* 64 (12), 2021.** <https://arxiv.org/abs/1803.09010>. The sibling discipline for datasets; useful lens for chapter 06 § 2 and the mod-110 eval-set documentation.
- **Hugging Face — Model Cards.** <https://huggingface.co/docs/hub/en/model-cards>. The widely-used ecosystem shape for model-card metadata.
- **Anthropic — Model and system cards.** <https://www.anthropic.com/transparency>. Anthropic's published cards and transparency hub; reference shapes for chapter 06.
- **OpenAI — System cards.** <https://openai.com/safety/>. OpenAI's published system cards (GPT-4, GPT-4o, o-series, and successor releases).
- **Google DeepMind — Model cards and responsibility reports.** <https://deepmind.google/responsibility/>. DeepMind's reporting hub.
- **Meta — Llama model cards.** <https://ai.meta.com/llama/>. Llama family cards.

<!-- needs-research: on the next research cycle, re-verify the Anthropic / OpenAI / Google DeepMind / Meta transparency pages — section names, URLs, and the current latest-published cards evolve per release; refresh the links above and sync chapter 06's "provider-published cards" reference list to the current set. -->

## NIST AI Risk Management Framework

Chapter 07's primary framework for United States / internal-governance alignment.

- **NIST AI Risk Management Framework (AI 100-1, January 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. The core RMF document — Govern / Map / Measure / Manage functions.
- **NIST — AI RMF Playbook.** <https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook>. Suggested actions per RMF subcategory; chapter 07's mapping references these action ids directly.
- **NIST — Generative AI Profile (AI 600-1, July 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. GenAI-specific actions layered on the base RMF functions; the dominant lens for app-side eval mapping.
- **NIST — Trustworthy and Responsible AI Resource Center.** <https://airc.nist.gov/>. The umbrella hub; useful landing page for refresh tracking.
- **NIST — AI Standards hub.** <https://www.nist.gov/artificial-intelligence/ai-standards>. Standards-coordination work the Playbook feeds.

## EU AI Act

Chapter 07's primary framework for EU regulatory alignment.

- **Regulation (EU) 2024/1689 — The AI Act (official text, OJ L 2024/1689, 12 July 2024).** <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. The Regulation itself; the primary source. Article 9 (risk management), 10 (data governance), 12 (record-keeping), 13 (transparency), 14 (human oversight), 15 (accuracy / robustness / cybersecurity), 51 – 55 (GPAI), 61 (post-market monitoring), 72 (post-market monitoring plan), 73 (serious-incident reporting).
- **European Commission — AI Act implementation hub.** <https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai>. Entry point for implementation guidance, delegated acts, and timelines.
- **European Commission — AI Office.** <https://digital-strategy.ec.europa.eu/en/policies/ai-office>. The AI Office governs GPAI provider obligations; refresh-tracking channel for Articles 51 – 55.
- **European Commission — General-Purpose AI Code of Practice.** <https://digital-strategy.ec.europa.eu/en/policies/ai-code-practice>. The Code of Practice for GPAI model providers (Article 56); evolving document.
- **CEN-CENELEC JTC 21 — AI standards.** <https://www.cencenelec.eu/areas-of-work/cen-cenelec-topics/artificial-intelligence/>. The European standards track for AI Act harmonised standards.
- **European Commission — AI Act compliance checker.** <https://artificialintelligenceact.eu/assessment/eu-ai-act-compliance-checker/>. Community-maintained scoping tool; useful first-pass for whether a surface is in-scope.

<!-- needs-research: on the next research cycle, re-verify EU AI Act GPAI obligation applicability dates (Articles 51 – 55 apply from 2025-08-02 for new models placed on the market; 2027-08-02 for models already on the market as of 2025-08-02; high-risk system obligations under Chapter III apply from 2026-08-02), the current state of harmonised standards publication, and the Code of Practice version. Refresh chapter 07 and exercise-05 accordingly. -->

## ISO/IEC 42001 and the ISO AI standards family

Chapter 07's primary framework for international / audit alignment.

- **ISO/IEC 42001:2023 — Information technology — Artificial intelligence — Management system.** <https://www.iso.org/standard/81230.html>. The AI management-system standard — Clauses 4 (context), 5 (leadership), 6 (planning), 7 (support), 8 (operation), 9 (performance evaluation), 10 (improvement); Annex A controls. Paywalled; chapter 07's mapping references clauses by number.
- **ISO/IEC 42005:2025 — AI system impact assessment.** <https://www.iso.org/standard/44545.html>. Companion standard for impact assessment of AI systems.
- **ISO/IEC 23894:2023 — Information technology — Artificial intelligence — Guidance on risk management.** <https://www.iso.org/standard/77304.html>. Risk-management guidance applied to AI; sibling to ISO 42001.
- **ISO/IEC 5259 series — Data quality for analytics and ML.** <https://www.iso.org/standard/81088.html>. Standards for data quality; useful cross-reference for mod-110 and mod-109 practices.
- **ISO/IEC 38507:2022 — Governance implications of the use of AI by organizations.** <https://www.iso.org/standard/56641.html>. Governance-level standard; useful cross-reference for the ai-governance-analyst peer.
- **ISO/IEC JTC 1/SC 42 — Artificial intelligence.** <https://www.iso.org/committee/6794475.html>. The subcommittee publishing the standards above; useful for refresh tracking.
- **BSI — Introduction to ISO/IEC 42001.** <https://www.bsigroup.com/en-GB/products-and-services/standards/iso-iec-420012023-artificial-intelligence-management-systems/>. One of the broad vendor-neutral overviews; useful summary for teams approaching 42001 for the first time.

## Related standards and emerging regulation

Cross-reference for teams whose org posture extends beyond the three primary frameworks.

- **ISO/IEC 27001:2022 — Information security management systems.** <https://www.iso.org/standard/27001>. The sibling management-system standard; the ISO 42001 clause structure mirrors it, and most orgs doing 42001 also do 27001.
- **ISO/IEC 27701 — Privacy information management.** <https://www.iso.org/standard/71670.html>. Privacy management; useful when the eval program handles personal data in traces or evaluation sets.
- **SOC 2 — AICPA Trust Services Criteria.** <https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2>. The attestation most SaaS vendors on the chapter 04 matrix carry.
- **UK AI Safety Institute — Inspect framework.** <https://inspect.aisi.org.uk/>. Agent / safety eval harness commonly used by peer teams; cross-reference from mod-103 and mod-108.
- **UK — Pro-innovation approach to AI regulation.** <https://www.gov.uk/government/publications/ai-regulation-a-pro-innovation-approach>. UK's regulatory approach; useful for UK-exposed deployments.
- **Singapore IMDA — Model AI Governance Framework.** <https://www.pdpc.gov.sg/-/media/files/pdpc/pdf-files/resource-for-organisation/ai/sgmodelaigovframework2.pdf>. One of the earliest voluntary governance frameworks; useful cross-reference.
- **Canada — Artificial Intelligence and Data Act (AIDA).** <https://ised-isde.canada.ca/site/innovation-better-canada/en/artificial-intelligence-and-data-act>. Canadian regulatory track; useful for North American deployments.
- **US — Executive Order 14110 (Safe, Secure, and Trustworthy AI).** <https://www.whitehouse.gov/briefing-room/presidential-actions/2023/10/30/executive-order-on-the-safe-secure-and-trustworthy-development-and-use-of-artificial-intelligence/>. The Biden EO on AI; superseded in parts by later actions but structurally relevant to US federal postures.
- **US — NAIAC and OSTP AI Bill of Rights blueprint.** <https://www.whitehouse.gov/ostp/ai-bill-of-rights/>. The AI Bill of Rights framing; useful cross-reference for the governance peer's language.

<!-- needs-research: on the next research cycle, re-verify the US executive-order landscape (EOs supersede frequently), the UK regulatory posture (White Paper updates), and the Canadian AIDA status; refresh the above URLs and chapter 07 mapping accordingly. -->

## Data residency, DPAs, and cross-border transfer

Chapter 04's `data_residency` axis depth references.

- **EU — General Data Protection Regulation (Regulation (EU) 2016/679).** <https://eur-lex.europa.eu/eli/reg/2016/679/oj>. The reference text for cross-border transfer rules affecting EU-hosted deployments.
- **European Data Protection Board (EDPB) — Guidelines on international data transfers.** <https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines_en>. The current state of the SCC / adequacy / transfer-impact-assessment landscape.
- **EU — Data Act (Regulation (EU) 2023/2854).** <https://eur-lex.europa.eu/eli/reg/2023/2854/oj>. Companion regulation that touches cloud switching and data portability — relevant to chapter 04's vendor-lock-in axis.
- **AWS, Azure, GCP — regional offerings for AI / ML.** Cross-cloud residency maps vary; verify current region support against each cloud's console. (Vendor-provided tables are maintained; URLs omitted to avoid citing a stale page.)

## Vendor incident learning and public post-mortems

Cross-reference for exercise 04's "learn from a real incident" shape.

- **Anthropic — status page.** <https://status.anthropic.com/>. Historical incidents on the Anthropic platform; useful source of real post-mortems with public summaries.
- **OpenAI — status page.** <https://status.openai.com/>. The OpenAI equivalent.
- **Google Cloud — status dashboard.** <https://status.cloud.google.com/>. Includes Vertex AI incidents.
- **AWS — Service Health Dashboard.** <https://health.aws.amazon.com/health/status>. Bedrock incidents published here.
- **Internet Robustness Research — public post-mortem index.** <https://github.com/danluu/post-mortems>. Dan Luu's curated post-mortem list; useful source for shape-of-postmortem reading when authoring exercise 04.

## Peer curriculum interfaces

Chapter 03's six delegation contracts and chapter 07's framework mapping bind to these peer tracks.

- **`model-evaluation-engineer-learning` (Model-Development family, level 30).** The peer that owns offline benchmark suites, judge-model training, dangerous-capability evals, and MLPerf-shaped serving benchmarks. Chapter 03 § Contract 1 and chapter 07 § GPAI obligations reference this track.
- **`ai-risk-engineer-learning` (Governance family, level 30).** The peer that owns the harm model, adversary personas, red-team data, and dangerous-capability escalation. Chapter 03 § Contract 2.
- **`ai-evaluation-engineer-learning` (Governance family, level 25).** The release-assurance peer; the first consumer of chapter 06's card slice and chapter 02's gate outcomes. Chapter 03 § Contract 3.
- **`ai-governance-analyst-learning`.** Org-wide AI policy, third-party attestations, vendor risk assessments, regulator interface. Chapter 03 § Contract 4; chapter 07's primary partner for framework interpretation.
- **`ai-infra-security-learning` (Infrastructure family, level 30).** Adversarial-eval infrastructure; judge / evaluator supply chain; trace-storage security. Chapter 03 § Contract 5.
- **`ai-infra-mlops-learning` / `ai-infra-ml-platform-learning` (Infrastructure family).** CI / CD integration surface; canary / ramp machinery; trace-collection substrate; deployment-lineage service. Chapter 03 § Contract 6.
- **`ai-infra-performance-learning` (Infrastructure family).** Fleet-altitude serving — kernels, batching, KV-cache, GPU-hours per token. Cross-reference from mod-111 chapter 07.
- **`llm-application-developer-learning` / `agentic-ai-engineer-learning`.** Product-team peers whose prompts, chains, and agent authoring feed the lineage keys the eval program's gates read.

## Prior modules this module composes over

Every chapter in mod-112 reads *down* into one or more of the eleven earlier modules.

- **mod-101-product-shaped-eval-foundations.** Definition of per-surface quality; the surface roster chapter 02 builds the gate map on.
- **mod-102-trace-instrumentation.** OTel-GenAI span attributes; the evidence base for every online gate; the delegation-contract target for `ai-infra-mlops`.
- **mod-103-trajectory-and-tool-eval.** Trajectory scores; the gate at ramp for agentic surfaces; card-slice § 2 for tool use.
- **mod-104-llm-as-judge-in-product.** Rubrics + calibration; the evidence base for offline gates; the delegation contract with `model-evaluation-engineer` on judge overlap.
- **mod-105-rag-eval-app-layer.** Retrieval + faithfulness scores; the offline gate for RAG surfaces; card-slice retrieval section.
- **mod-106-eval-gated-cicd.** Replay bundle + gate result; the mechanism chapter 02's offline gate binds to.
- **mod-107-online-eval-and-regression.** Online drift + rollback signals; the canary / ramp / post-ship machinery.
- **mod-108-app-safety-and-guardrails-eval.** Findings + OWASP tags; chapter 02's safety-row threshold; chapter 03's contract with `ai-risk-engineer` and `ai-infra-security`.
- **mod-109-human-review-workflows.** Reviewer-labelled cases; the gate at offline; the delegation-contract adjudication shape.
- **mod-110-eval-data-platform-slice.** Store, schemas, lineage; the read layer every gate reads from; the build-vs-buy substrate for chapter 04.
- **mod-111-cost-latency-quality-tradeoff.** Multi-objective report; the evidence bundle every shipping decision the gate authorises publishes; the delegation contract with `model-evaluation-engineer` on model-swap regression.

<!-- needs-research: on the next research cycle, re-verify the eval-platform vendor URLs (vendor renames / deprecations happen roughly quarterly), the EU AI Act delegated-act and harmonised-standards status, the NIST AI RMF Playbook action-id revisions, and the ISO/IEC 42001 adjacent-standard list. Refresh the chapter 04 component roster, the chapter 07 provision mapping headlines, and the resources above accordingly. -->
