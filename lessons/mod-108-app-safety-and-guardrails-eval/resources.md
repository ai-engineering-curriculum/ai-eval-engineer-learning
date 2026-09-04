# Resources for mod-108-app-safety-and-guardrails-eval (App-Side Safety and Guardrail-Effectiveness Evaluation)

Primary references cited across the chapters and exercises. Prefer official docs and original papers over blog posts — the guardrail-vendor surfaces, the injection-benchmark repos, and the OWASP / ATLAS / SAIF pages all update on their own cadences and the summaries drift. Read from the URL each time you plan to build against a specific behaviour or map a finding.

Where the source contains harmful example prompts (HarmBench, some AdvBench derivatives, some AgentDojo tasks), consume by reference per the payload-management contract in chapter 02 — do not check the prompts into your repo.

## Jailbreak-resistance benchmarks and attack methods

- **HarmBench — paper.** Mazeika, M., Phan, L., Yin, X., Zou, A., Wang, Z., Mu, N., Sakhaee, E., Li, N., Basart, S., Li, B., Forsyth, D., Hendrycks, D. "HarmBench: A Standardized Evaluation Framework for Automated Red Teaming and Robust Refusal." *ICML* 2024. <https://arxiv.org/abs/2402.04249>. The benchmark shape (behaviour × attack matrix), the ASR metric, and the default HarmBench-Cls judge referenced in chapter 02 and exercise 01.
- **HarmBench — repo.** <https://github.com/centerforaisafety/HarmBench>. Reference implementations of the runner, the classifier, and the attack families; the `Target` adapter interface exercise 01 wraps.
- **HarmBench — project page.** <https://www.harmbench.org>. Behaviour categories and result tables; access acknowledgement page for the gated behaviours.
- **AdvBench — paper.** Zou, A., Wang, Z., Kolter, J. Z., Fredrikson, M. "Universal and Transferable Adversarial Attacks on Aligned Language Models." *arXiv:2307.15043*, 2023. <https://arxiv.org/abs/2307.15043>. The original GCG (Greedy Coordinate Gradient) adversarial-suffix attack; AdvBench is its evaluation set. Cited in chapter 02's attack-family survey.
- **PAIR — paper.** Chao, P., Robey, A., Dobriban, E., Hassani, H., Pappas, G. J., Wong, E. "Jailbreaking Black Box Large Language Models in Twenty Queries." *arXiv:2310.08419*, 2023. <https://arxiv.org/abs/2310.08419>. Automated iterative attacker-LLM rewriter; the black-box attack family exercise 01's stretch goal wires in.
- **TAP — paper.** Mehrotra, A., Zampetakis, M., Kassianik, P., Nelson, B., Anderson, H., Singer, Y., Karbasi, A. "Tree of Attacks: Jailbreaking Black-Box LLMs Automatically." *NeurIPS* 2024. <https://arxiv.org/abs/2312.02119>. Tree-of-Attacks-with-Pruning; the second black-box automated attack HarmBench scores.
- **PAP — paper.** Zeng, Y., Lin, H., Zhang, J., Yang, D., Jia, R., Shi, W. "How Johnny Can Persuade LLMs to Jailbreak Them: Rethinking Persuasion to Challenge AI Safety by Humanizing LLMs." *ACL* 2024. <https://arxiv.org/abs/2401.06373>. Persuasive Adversarial Prompts; a strong-transfer attack family for chapter 02.
- **AutoDAN — paper.** Liu, X., Xu, N., Chen, M., Xiao, C. "AutoDAN: Generating Stealthy Jailbreak Prompts on Aligned Large Language Models." *ICLR* 2024. <https://arxiv.org/abs/2310.04451>. Genetic-algorithm-based jailbreak generation.
- **Best-of-N Jailbreaking — paper.** Hughes, J., Price, S., Lynch, A., Schaeffer, R., Barez, F., Koyejo, S., Sleight, H., Jones, E., Perez, E., Sharma, M. "Best-of-N Jailbreaking." *arXiv:2412.03556*, 2024. <https://arxiv.org/abs/2412.03556>. Sampling-based attack against reasoning models; a modern column not in the 2024 HarmBench snapshot.
- **XSTest — paper.** Röttger, P., Kirk, H. R., Vidgen, B., Attanasio, G., Bianchi, F., Hovy, D. "XSTest: A Test Suite for Identifying Exaggerated Safety Behaviours in Large Language Models." *NAACL* 2024. <https://arxiv.org/abs/2308.01263>. Over-refusal / exaggerated-safety test suite; the reference for chapter 06 and the adversarial-benign subset in exercise 04.
- **XSTest — repo.** <https://github.com/paul-rottger/exaggerated-safety>. The test prompts and grading scripts.
- **OR-Bench — paper.** Cui, J., Chiang, W.-L., Stoica, I., Hsieh, C.-J. "OR-Bench: An Over-Refusal Benchmark for Large Language Models." *arXiv:2405.20947*, 2024. <https://arxiv.org/abs/2405.20947>. Large-scale over-refusal benchmark used alongside XSTest in chapter 06.
- **OR-Bench — project page and dataset.** <https://huggingface.co/datasets/bench-llm/or-bench>. The dataset artefact and the leaderboard.
- **JailbreakBench — paper.** Chao, P., Debenedetti, E., Robey, A., Andriushchenko, M., Croce, F., Sehwag, V., Dobriban, E., Flammarion, N., Pappas, G. J., Tramer, F., Hassani, H., Wong, E. "JailbreakBench: An Open Robustness Benchmark for Jailbreaking Large Language Models." *NeurIPS Datasets and Benchmarks* 2024. <https://arxiv.org/abs/2404.01318>. Complementary open benchmark; useful as a cross-check on HarmBench numbers.
- **JailbreakBench — project page.** <https://jailbreakbench.github.io>. Leaderboard and dataset.
- **StrongREJECT — paper.** Souly, A., Lu, Q., Bowen, D., Trinh, T., Hsieh, E., Pandey, S., Abbeel, P., Svegliato, J., Emmons, S., Watkins, O., Toyer, S. "A StrongREJECT for Empty Jailbreaks." *NeurIPS* 2024. <https://arxiv.org/abs/2402.10260>. A tighter judge for jailbreak success that avoids inflating ASR on vacuous "jailbreaks"; referenced in chapter 02's judge discussion.
- **DAN and community jailbreak taxonomy — paper.** Shen, X., Chen, Z., Backes, M., Shen, Y., Zhang, Y. "Do Anything Now: Characterizing and Evaluating In-The-Wild Jailbreak Prompts on Large Language Models." *CCS* 2024. <https://arxiv.org/abs/2308.03825>. Ecosystem study of hand-crafted jailbreaks; the "human-written" attack family HarmBench scores against.

## Attack tooling and red-team frameworks

- **garak — repo.** <https://github.com/NVIDIA/garak>. NVIDIA's LLM vulnerability scanner; probe library covers encoding attacks, translation attacks, and jailbreak variants. Chapter 02's alternate attack source and exercise 01's `garak` integration stretch goal.
- **garak — documentation.** <https://reference.garak.ai/>. Probes, detectors, and buffs; the extension surface for a custom probe.
- **PyRIT — repo.** <https://github.com/Azure/PyRIT>. Microsoft's Python Risk Identification Toolkit for generative AI; orchestrators for multi-turn red-teaming and per-attack scorer wiring. Chapter 02's third attack-orchestration reference.
- **PyRIT — documentation.** <https://azure.github.io/PyRIT/>. Framework guide and example notebooks.
- **DeepEval — red-teaming module.** <https://docs.confident-ai.com/docs/red-teaming-introduction>. Open-source red-team scenario runner; useful as a fourth attack-orchestration option.
- **Promptfoo — LLM security testing.** <https://www.promptfoo.dev/docs/red-team/>. YAML-configured red-team scenarios; simple entry point for teams that already run Promptfoo evals.
- **Meta's Purple Llama and CyberSec Eval.** <https://ai.meta.com/llama/purple-llama/> and <https://github.com/meta-llama/PurpleLlama>. Cyber-attack evaluation suite for LLM-generated code; useful cross-reference for the LLM04 / LLM06 mapping in chapter 07.

## Prompt-injection benchmarks and defence research

- **AgentDojo — paper.** Debenedetti, E., Zhang, J., Balunović, M., Beurer-Kellner, L., Fischer, M., Tramèr, F. "AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents." *NeurIPS Datasets and Benchmarks* 2024. <https://arxiv.org/abs/2406.13352>. The benchmark shape (user-goal / injection-goal split; `utility × security` scorecard) chapter 03 and exercise 02 target.
- **AgentDojo — repo.** <https://github.com/ethz-spylab/agentdojo>. Reference implementation and the pipeline-element interface exercise 02's `agent_adapter.py` wraps.
- **InjecAgent — paper.** Zhan, Q., Liang, Z., Ying, Z., Kang, D. "InjecAgent: Benchmarking Indirect Prompt Injections in Tool-Integrated Large Language Model Agents." *ACL Findings* 2024. <https://arxiv.org/abs/2403.02691>. Tool-abuse-focused indirect-injection benchmark; the InjecAgent cross-check in chapter 03 and exercise 02's stretch goal.
- **InjecAgent — repo.** <https://github.com/uiuc-kang-lab/InjecAgent>. Reference implementation.
- **Simon Willison — "Prompt injection: what's the worst that can happen?"** <https://simonwillison.net/2023/Apr/14/worst-that-can-happen/>. The reference articulation of the direct / indirect distinction that chapter 03 uses as vocabulary.
- **Indirect Prompt Injection — paper.** Greshake, K., Abdelnabi, S., Mishra, S., Endres, C., Holz, T., Fritz, M. "Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection." *AISec* 2023. <https://arxiv.org/abs/2302.12173>. The foundational academic treatment of indirect prompt injection; chapter 03's threat-shape citation.
- **StruQ — structured queries defence.** Chen, S., Piet, J., Sitawarin, C., Wagner, D. "StruQ: Defending Against Prompt Injection with Structured Queries." *USENIX Security* 2025. <https://arxiv.org/abs/2402.06363>. Structured-query defence referenced in chapter 03's defence-layer survey.
- **Spotlighting — Microsoft.** Hines, K., Lopez, G., Hall, M., Zarfati, F., Zunger, Y., Kiciman, E. "Defending Against Indirect Prompt Injection Attacks With Spotlighting." *arXiv:2403.14720*, 2024. <https://arxiv.org/abs/2403.14720>. Datamarking / encoding-based defences for indirect injection.
- **Instruction Hierarchy — OpenAI.** Wallace, E., Xiao, K., Leike, R., Weng, L., Heidecke, J., Beutel, A. "The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions." *arXiv:2404.13208*, 2024. <https://arxiv.org/abs/2404.13208>. Training-time defence for prioritising the developer / user / third-party hierarchy; the trust-tier vocabulary chapter 03 references.
- **Prompt Injections Against GPT-4 — DeepMind analysis.** Toyer, S., Watkins, O., Mendes, E. A., Svegliato, J., Bailey, L., Wang, T., Ong, I., Elmaaroufi, K., Abbeel, P., Darrell, T., Ritter, A., Russell, S. "Tensor Trust: Interpretable Prompt Injection Attacks from an Online Game." *ICLR* 2024. <https://arxiv.org/abs/2311.01011>. Human-generated prompt-injection corpus from a public online game; useful ecosystem-shaped attack source.
- **Anthropic — prompt injection risks.** <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/system-prompts>. Provider guidance on system-prompt shape and instruction robustness; referenced in chapter 03's defence discussion.
- **OpenAI — safety best practices.** <https://platform.openai.com/docs/guides/safety-best-practices>. Provider guidance on injection, content filtering, and moderation for the app layer.

## Tool-abuse and agent-safety research

- **ToolEmu — paper.** Ruan, Y., Dong, H., Wang, A., Pitis, S., Zhou, Y., Ba, J., Dubois, Y., Maddison, C. J., Hashimoto, T. "Identifying the Risks of LM Agents with an LM-Emulated Sandbox." *ICLR* 2024. <https://arxiv.org/abs/2309.15817>. LM-emulated tool sandbox for surfacing agent failure modes; the emulated-tool pattern chapter 04 references.
- **ToolEmu — repo.** <https://github.com/ryoungj/ToolEmu>. Reference implementation.
- **AgentBench — paper.** Liu, X., Yu, H., Zhang, H., Xu, Y., Lei, X., Lai, H., Gu, Y., Ding, H., Men, K., Yang, K., Zhang, S., Deng, X., Zeng, A., Du, Z., Zhang, C., Shen, S., Zhang, T., Su, Y., Sun, H., Huang, M., Dong, Y., Tang, J. "AgentBench: Evaluating LLMs as Agents." *ICLR* 2024. <https://arxiv.org/abs/2308.03688>. Broad agent evaluation benchmark; useful cross-reference for the utility side of chapter 03's `utility × security` scorecard.
- **Canary tokens.** <https://canarytokens.org/>. Thinkst's canary token service; the operational reference for the canary-token pattern chapter 04 walks. Documentation on token generation, detection scope, and integration with SIEM.
- **AgentHarm — paper.** Andriushchenko, M., Souly, A., Dziemian, M., Duenas, D., Lin, M., Wang, J., Hendrycks, D., Zou, A., Kolter, Z., Fredrikson, M., Winsor, E., Wynne, J., Gal, Y., Davies, X. "AgentHarm: A Benchmark for Measuring Harmfulness of LLM Agents." *ICLR* 2025. <https://arxiv.org/abs/2410.09024>. Harmfulness benchmark for tool-using agents; complementary to AgentDojo / InjecAgent for tool-abuse assessment.
- **AgentHarm — dataset.** <https://huggingface.co/datasets/ai-safety-institute/AgentHarm>. The public artefact.
- **Google — Agent Safety Framework.** <https://safety.google/intl/en_us/cybersecurity-advancements/saif/>. SAIF's agent-safety guidance; referenced alongside chapter 04's sandbox pattern.

## Guardrail stack

- **Llama Guard 3 — model card.** <https://huggingface.co/meta-llama/Llama-Guard-3-8B>. The classifier the OSS guardrail in exercise 04 targets; hazard categories and threshold guidance.
- **Llama Guard — paper.** Inan, H., Upasani, K., Chi, J., Rungta, R., Iyer, K., Mao, Y., Tontchev, M., Hu, Q., Fuller, B., Testuggine, D., Khabsa, M. "Llama Guard: LLM-based Input-Output Safeguard for Human-AI Conversations." *arXiv:2312.06674*, 2023. <https://arxiv.org/abs/2312.06674>. The design paper for the input / output moderator family.
- **MLCommons AI Safety hazard taxonomy.** Vidgen, B., Agrawal, A., Ahmed, A. M., Akinwande, V., Al-Nuaimi, N., Alfaraj, N., Alhajjar, E., Aroyo, L., Bavalatti, T., Blili-Hamelin, B., et al. "Introducing v0.5 of the AI Safety Benchmark from MLCommons." *arXiv:2404.12241*, 2024. <https://arxiv.org/abs/2404.12241>. The hazard taxonomy Llama Guard 3 maps to; referenced in chapter 05's category-mapping table stretch goal.
- **NeMo Guardrails — documentation.** <https://docs.nvidia.com/nemo/guardrails/latest/index.html>. NVIDIA's programmable rails framework; Colang policies and the dialog-manager pattern chapter 05 surveys.
- **NeMo Guardrails — repo.** <https://github.com/NVIDIA/NeMo-Guardrails>. OSS source and example configs.
- **Guardrails AI — documentation.** <https://docs.guardrailsai.com/>. Validator library for LLM outputs; chapter 05's second OSS reference.
- **Guardrails AI — repo.** <https://github.com/guardrails-ai/guardrails>. OSS source and the validator hub.
- **OpenAI Moderation — documentation.** <https://platform.openai.com/docs/guides/moderation>. The `omni-moderation-latest` model, categories, and thresholds referenced in exercise 04's hosted-guardrail option.
- **OpenAI Moderation API — model card.** <https://cdn.openai.com/papers/omni_moderation_api.pdf>. Model card for the omni-moderation model.
- **Azure AI Content Safety — documentation.** <https://learn.microsoft.com/en-us/azure/ai-services/content-safety/>. Categories, severity levels, prompt-shields, and jailbreak detection; alternate hosted-guardrail option for exercise 04.
- **Google Model Armor — documentation.** <https://cloud.google.com/security-command-center/docs/model-armor-overview>. Google's model-safety filter; the third hosted-guardrail option cited in chapter 05.
- **Presidio — documentation.** <https://microsoft.github.io/presidio/>. Microsoft's PII detection and anonymisation toolkit; chapter 05's PII-oriented guardrail reference and a common component of tool-argument checkers in chapter 04.
- **Presidio — repo.** <https://github.com/microsoft/presidio>. OSS source.
- **Perspective API — documentation.** <https://perspectiveapi.com/>. Jigsaw / Google's toxicity classifier; historically common as an output guardrail component.
- **AWS Bedrock Guardrails — documentation.** <https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html>. Bedrock's built-in guardrail configuration; useful cross-reference in AWS-hosted deployments.
- **Prompt Guard — Meta.** <https://huggingface.co/meta-llama/Prompt-Guard-86M>. Small classifier for prompt-injection / jailbreak detection at the input layer; chapter 05's lightweight input-classifier option.

## Provider usage policies

- **OpenAI — usage policies.** <https://openai.com/policies/usage-policies>. The provider policy against which the OpenAI-hosted surface's raw model is scored (peer-owned); the chapter 06 reconciliation cites your app's policy as a subset of the provider policy.
- **Anthropic — usage policy.** <https://www.anthropic.com/legal/aup>. Provider policy for Anthropic-hosted surfaces.
- **Google — generative AI prohibited use policy.** <https://policies.google.com/terms/generative-ai/use-policy>. Provider policy for Google-hosted surfaces (Gemini, Vertex AI).
- **Meta — Llama 3 Acceptable Use Policy.** <https://llama.meta.com/llama3/use-policy/>. Provider policy for Llama-hosted surfaces.
- **Mistral — terms of service.** <https://mistral.ai/terms/>. Provider policy for Mistral-hosted surfaces.

## OWASP LLM Top 10 and OWASP GenAI Security Project

- **OWASP Top 10 for LLM Applications — 2025.** <https://genai.owasp.org/llm-top-10/>. The current LLM Top 10 categories (`LLM01`–`LLM10`) chapter 07 and exercise 05 map findings against. Read from the URL each mapping cycle — categories evolve version to version.
- **OWASP Top 10 for LLM Applications — 2025 PDF.** <https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/>. The versioned document for the current release.
- **OWASP GenAI Security Project — landing page.** <https://genai.owasp.org/>. Working groups, adjacent guides (LLM AI Cybersecurity & Governance Checklist, LLM Applications and Generative AI Security Solutions Landscape), and version history.
- **OWASP GenAI Security Project — GitHub.** <https://github.com/OWASP/www-project-top-10-for-large-language-model-applications>. The source repo; category drafts and PRs give early warning of category changes.

<!-- needs-research: on the next research cycle, re-verify the OWASP LLM Top 10 (2025) category slugs and text at https://genai.owasp.org/llm-top-10/ — the list is versioned and category text updates roughly annually; sync `mappers/owasp_llm_top10.py` in exercise 05 to the current release. -->

## MITRE ATLAS

- **MITRE ATLAS — matrix.** <https://atlas.mitre.org/matrices/ATLAS>. The tactics × techniques matrix chapter 07 tags findings against.
- **MITRE ATLAS — techniques.** <https://atlas.mitre.org/techniques/>. Individual technique pages with descriptions and mitigations; `AML.T0051 Prompt Injection` (and sub-techniques `.001 Direct`, `.002 Indirect`), `AML.T0057 LLM Data Leakage`, `AML.T0025 Exfiltration via Cyber Means`, and others chapter 07 cites.
- **MITRE ATLAS — case studies.** <https://atlas.mitre.org/studies>. Public case studies of adversarial ML incidents; the sanity-check reference chapter 07 recommends.
- **MITRE ATLAS — data repo.** <https://github.com/mitre-atlas/atlas-data>. STIX / JSON representations of the matrix; useful when scripting mappers.
- **MITRE ATT&CK — main site.** <https://attack.mitre.org/>. The parent framework; ATLAS reuses the tactic / technique / procedure vocabulary. Useful cross-reference for the operational-security side of a finding.

<!-- needs-research: on the next research cycle, re-verify the MITRE ATLAS technique catalogue at https://atlas.mitre.org — techniques and sub-techniques are added over time; sync `mappers/mitre_atlas.py` in exercise 05 to the current snapshot. -->

## Google SAIF

- **Google Secure AI Framework — overview.** <https://safety.google/intl/en_us/cybersecurity-advancements/saif/>. The six-axis framework chapter 07 cites in report summaries.
- **SAIF Risk Map.** <https://saif.google/secure-ai-framework/risk-map>. Interactive risk-map linking SAIF axes to specific risks and mitigations.
- **SAIF — introduction paper.** Google. "Introducing Google's Secure AI Framework." 2023. <https://blog.google/technology/safety-security/introducing-googles-secure-ai-framework/>. The launch post that names the six axes.

## NIST AI RMF and the Generative AI Profile

- **NIST AI Risk Management Framework (AI 100-1).** <https://www.nist.gov/itl/ai-risk-management-framework>. The Govern / Map / Measure / Manage functions chapter 07 maps findings against on regulated deployments.
- **NIST AI RMF — full document.** <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf>. The 2023 published framework.
- **NIST AI RMF Generative AI Profile (AI 600-1).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. The GenAI-specific action set chapter 07 tags on regulated findings.
- **NIST AI RMF Playbook.** <https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook>. Suggested actions per function; the `Measure-*` and `Manage-*` action ids exercise 05's `mappers/nist_rmf.py` targets.

<!-- needs-research: on the next research cycle, re-verify the AI RMF Playbook actions relevant to safety eval at https://airc.nist.gov and check for updates to the GenAI Profile; the action id set changes as NIST publishes revisions. -->

## Adjacent standards and regulation

- **ISO/IEC 42001:2023 — AI management systems.** <https://www.iso.org/standard/81230.html>. AI-management-system controls that reference continuous evaluation and change management. Paywalled; a summary is at <https://www.iso.org/artificial-intelligence/artificial-intelligence>.
- **ISO/IEC 23894:2023 — AI risk management guidance.** <https://www.iso.org/standard/77304.html>. Companion risk-management guidance to 42001.
- **EU Artificial Intelligence Act (Regulation (EU) 2024/1689).** <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. High-risk system obligations (Article 9 risk management, Article 15 accuracy / robustness / cybersecurity, Article 72 post-market monitoring, Article 73 serious-incident reporting); the safety-eval report and runbook are load-bearing artefacts on a high-risk deployment.
- **UK AI Safety Institute — evaluations.** <https://www.aisi.gov.uk/work>. Public evaluations of frontier-model safety; useful benchmark ceiling for dangerous-capability discussions with the model-eval-eng peer.
- **US AI Safety Institute — model evaluations program.** <https://www.nist.gov/aisi>. NIST's AI Safety Institute; the US counterpart.

## Peer and adjacent frameworks worth reading

- **Anthropic — Responsible Scaling Policy.** <https://www.anthropic.com/rsp>. Provider RSP; a reference for how a frontier lab structures capability-level safety commitments the app-surface eval hands off to.
- **OpenAI — Preparedness Framework.** <https://openai.com/safety/preparedness>. OpenAI's dangerous-capability tracking; the peer counterpart the model-eval-eng escalation lands against.
- **Google DeepMind — Frontier Safety Framework.** <https://deepmind.google/discover/blog/updating-the-frontier-safety-framework/>. Google's capability-based safety framework.
- **AI Village — DEF CON generative-AI red-team report.** <https://aivillage.org/generative%20red%20team/generative-red-team/>. Public red-team exercise from DEF CON 31; a real-ecosystem attack shape useful when calibrating internal-suite realism.

## Peer curriculum interfaces

- **`ai-risk-engineer-learning` (Governance family, level 30)** — the peer that owns the harm model, adversary personas, and red-team data generation. Chapter 01 defines the delegation triangle; exercise 05 encodes escalations to this peer as machine-readable conditions.
- **`model-evaluation-engineer-learning` (Model-Development family, level 30)** — the peer that owns model-level dangerous-capability evals (weapons uplift, bio, cyber, autonomy). Chapter 07 and exercise 05 route escalations here when app-surface defences are ineffective.
- **`ai-evaluation-engineer-learning` (Governance family, level 25)** — the sign-off peer for release-gate and continuous-monitoring architecture; mod-112 formalises this hand-off, of which this module's report and runbook are inputs.
- **`agentic-ai-engineer-learning`** — the agent implementations whose surface the injection and tool-abuse evals target; the tool inventory chapter 04 depends on is co-owned with this peer.
- **`llm-application-developer-learning`** — the LLM application engineering whose prompt / chain / guardrail changes the mod-106 gate and mod-107 online loop are watching for regressions.

<!-- needs-research: on the next research cycle, re-verify current AgentDojo / InjecAgent / AgentHarm dataset URLs and repo layouts (agent-eval benchmarks move on a monthly cadence), refresh the Llama Guard / NeMo Guardrails / Guardrails AI / Presidio / Model Armor / Azure AI Content Safety documentation links, and re-check the OWASP LLM Top 10 (2025) release, the MITRE ATLAS technique catalogue, and the NIST AI RMF Playbook actions for updates. -->
