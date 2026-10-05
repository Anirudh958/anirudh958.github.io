---
layout: module
title: "Defending AI Systems — Building Secure AI Pipelines"
path_id: ai-security
module: 6
description: "The blue team module. After five modules of breaking things, this one teaches you to build AI systems that survive contact with the threat actors you've been studying. From input guardrails to supply chain integrity to the brand-new NVIDIA OpenShell runtime — this is defense engineering for AI systems as of October 2026, not the sanitized version from a compliance slide deck."
---

*By Anirudh Gupta Surisetty · Last updated October 2026 · ~30 min read · Prerequisite: Module 5 — AI-Powered Offensive Operations*

## Why Defense Is Harder Than Attack (And Why Most Teams Get It Wrong)

I'm going to tell you something that will sound backwards if you've been reading vendor marketing: **most AI security products protect against the wrong layer.**

Go look at any LLM firewall vendor's landing page. Count how many times they mention "prompt injection." Now count how many times they mention "MCP tool poisoning," "supply chain integrity," or "agent runtime sandboxing."

The ratio tells you everything. The industry is selling shields for the front door while the real threats come through the plumbing.

Here's the reality as of October 2026:

- **[OWASP ranks prompt injection as LLM01](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)** — the #1 vulnerability for LLM applications. They're right. But ranking doesn't mean it's the most *dangerous*. It's the most *common*.
- The attacks that actually destroyed organizations in 2026 — LiteLLM, JADEPUFFER, the Trivy compromise — operated at the supply chain and agent layers.
- **Only 21% of organizations** have a mature governance model for agentic AI ([Deloitte, 2026](https://www.deloitte.com/us/en/insights/topics/emerging-technologies/ai-agents-scaling-faster.html)).
- **Only 7.2%** have a named individual formally accountable for agent behavior ([Gravitee, 2026](https://www.gravitee.io/state-of-ai-agent-security)).
- **54% of organizations** experienced or suspected an AI agent security or data privacy incident in the past 12 months ([Gravitee, April 2026](https://www.gravitee.io/state-of-ai-agent-security)).

The gap between "we have AI in production" and "we have AI security in production" is a canyon. This module builds the bridge.

---

## 6.1 — The Defense Stack: A Layered Architecture

### Why Layers Matter

A single guardrail is a speedbump. A layered defense is a fortress. The fundamental principle hasn't changed since the 1990s — defense in depth — but the layers are completely different for AI systems.

### The AI Defense Stack (From Outside In)

```
┌─────────────────────────────────────────────────┐
│  Layer 7: Governance & Compliance               │
│  (Policies, accountability, audit, regulation)  │
├─────────────────────────────────────────────────┤
│  Layer 6: Supply Chain Integrity                │
│  (Model signing, SLSA, dependency verification) │
├─────────────────────────────────────────────────┤
│  Layer 5: Agent Runtime Security                │
│  (Sandboxing, tool permissions, action audit)   │
├─────────────────────────────────────────────────┤
│  Layer 4: Model Security                        │
│  (Weight integrity, extraction detection)       │
├─────────────────────────────────────────────────┤
│  Layer 3: Output Validation                     │
│  (PII filtering, toxicity, hallucination check) │
├─────────────────────────────────────────────────┤
│  Layer 2: Inference Security                    │
│  (Rate limiting, anomaly detection, logging)    │
├─────────────────────────────────────────────────┤
│  Layer 1: Input Validation                      │
│  (Prompt guards, injection detection, filters)  │
└─────────────────────────────────────────────────┘
```

> **Note:** these seven layers are a *defense* stack. They're numbered differently from the five-layer *attack surface* in Module 0, where Layer 5 is Infrastructure.

Most organizations implement Layer 1 and maybe Layer 3, then call it done. The breaches of 2026 exploited Layers 4–6 almost exclusively.

Here's the hard truth, and it's worth memorizing: *"Guardrails without protocol security produce polite chatbots that still exfiltrate data through a tool call. Protocol security without guardrails produces well-authenticated agents that still follow malicious instructions embedded in an artifact. You need both, plus runtime policy for high-risk actions."*

Let's build each layer.

---

## 6.2 — Layer 1: Input Validation for LLMs

### The Multi-Layer Guardrail Architecture

A single regex pattern catching "ignore previous instructions" stopped working in 2023. Modern input validation requires a cascade:

**Stage 1: Syntactic Filtering (Fast, Cheap)**

- Pattern matching for known injection signatures
- Encoding detection (Base64, ROT13, Unicode normalization attacks, JSFuck — remember the Manus AI bypass from Module 1)
- Input length limits and format validation
- Token budget enforcement

**Stage 2: Classifier-Based Detection (Moderate Cost)**

- Trained classifiers that detect injection attempts semantically, not syntactically
- These catch rephrased attacks that bypass keyword filters
- Tools: Lakera Guard ([acquired by Check Point](https://www.checkpoint.com/press-releases/check-point-acquires-lakera-to-deliver-end-to-end-ai-security-for-enterprises/), September 2025, for a [reported ~$300M](https://www.calcalistech.com/ctechnews/article/rj5bc1vige)), [Cisco AI Defense](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2025/m01/cisco-unveils-ai-defense-to-secure-the-ai-transformation-of-enterprises.html) (built on Cisco's [acquisition of Robust Intelligence](https://blogs.cisco.com/news/fortifying-the-future-of-security-for-ai-cisco-announces-intent-to-acquire-robust-intelligence)), Prompt Security ([acquired by SentinelOne](https://www.sentinelone.com/press/sentinelone-to-acquire-prompt-security-to-advance-genai-security/), August 2025)
- Key limitation: classifiers can be adversarially evaded — they're a layer, not a solution

**Stage 3: Semantic Analysis (Highest Accuracy, Highest Cost)**

- Use a second LLM to evaluate whether the input is attempting to override system instructions
- "Is this input asking the model to behave differently than its system prompt intends?"
- This catches sophisticated attacks that evade both pattern matching and classifiers
- The cost: you're running two LLM calls per request. Budget accordingly.

**Stage 4: Context-Aware Filtering (For Agentic Systems)**

- Evaluate inputs not just for injection, but for their relationship to the agent's tool permissions
- "This input mentions file paths — does the user have permission to trigger file operations?"
- This is where guardrails meet authorization, and most systems fail here completely

### What the OWASP Top 10 for LLMs Actually Says

The [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) (2025 edition, still current) maps the territory:

| Rank | Risk | Defense Priority |
|------|------|-----------------|
| LLM01 | Prompt Injection | Input validation (Layers 1-2) |
| LLM02 | Sensitive Information Disclosure | Output filtering (Layer 3) |
| LLM03 | Supply Chain | Supply chain integrity (Layer 6) |
| LLM04 | Data and Model Poisoning | Model security (Layer 4) |
| LLM05 | Improper Output Handling | Output validation (Layer 3) |
| LLM06 | Excessive Agency | Agent runtime security (Layer 5) |
| LLM07 | System Prompt Leakage | Input/output filtering |
| LLM08 | Vector and Embedding Weaknesses | RAG security |
| LLM09 | Misinformation | Output validation |
| LLM10 | Unbounded Consumption | Rate limiting (Layer 2) |

And OWASP's newer companion — the **[Top 10 for Agentic Applications for 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)**, published December 2025 — adds the agent-specific risks: agent goal hijack, tool misuse and exploitation, identity and privilege abuse, agentic supply chain vulnerabilities, unexpected code execution, memory and context poisoning, insecure inter-agent communication, cascading failures, human-agent trust exploitation, and rogue agents. If you're deploying agents, you need both lists.

### The Uncomfortable Truth About Input Validation

Here's what no vendor will tell you: **no input validation system achieves 100% detection of prompt injection**. The UK's National Cyber Security Centre [puts it plainly](https://www.ncsc.gov.uk/blog-post/prompt-injection-is-not-sql-injection): "As there is no inherent distinction between 'data' and 'instruction', it's very possible that prompt injection attacks may never be totally mitigated." A [2026 evaluation of prompt injection defenses](https://arxiv.org/abs/2604.23887) reached a practical version of the same conclusion: "Every defense that relied on the model to protect itself eventually broke. The only defense that held was output filtering."

This is not a bug to fix. It's a property of how transformer architectures process text. Defense in depth isn't optional — it's the only strategy that survives first contact with a determined attacker.

---

## 6.3 — Layer 5: Agent Runtime Security (The New Frontier)

### Why This Layer Didn't Exist 12 Months Ago

In 2025, "AI security" meant protecting against prompt injection and maybe checking outputs for PII. In 2026, agents happened. And agents changed everything.

When an AI system can:

- Read files
- Execute code
- Send emails
- Call APIs
- Access databases
- Interact with other agents

...every one of those capabilities is an attack surface. The agent's autonomy *is* the risk. The same autonomy that makes agents useful makes them dangerous.

The U.S. National Security Agency [stated in May 2026](https://www.nsa.gov/Press-Room/Press-Releases-Statements/Press-Release-View/Article/4496698/nsa-releases-security-design-considerations-for-ai-driven-automation-leveraging/) that MCP's *"rapid proliferation has outpaced the development of its security model."* They weren't wrong.

### The MCP Security Problem

Model Context Protocol (MCP) — the standard for how LLMs interact with external tools — was designed for functionality first, security second. The specification:

- **Makes authorization optional** — the [OAuth 2.1-based authorization framework](https://modelcontextprotocol.io/specification/latest/basic/authorization), added in the March 2025 revision, says "Authorization is OPTIONAL," and STDIO servers are told to pull credentials from the environment instead
- **Leaves tool-call decisions to the model** — nothing in the protocol stops the model from calling any tool it has been given
- **Treats tool descriptions as trusted** — which enables tool poisoning attacks (hidden instructions in tool descriptions that the AI reads but users can't see, [first demonstrated by Invariant Labs](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) in April 2025)

The project has since published [security best practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices) documenting risks like "confused deputy" vulnerabilities in MCP proxy servers. But the installed base is massive, and much of it predates these protections.

**Known MCP vulnerabilities (2025–2026):**

- **[CVE-2025-6514](https://nvd.nist.gov/vuln/detail/CVE-2025-6514)** (CVSS 9.6) — Critical flaw in mcp-remote allowing malicious servers to execute arbitrary commands on user machines
- **[CVE-2025-49596](https://nvd.nist.gov/vuln/detail/CVE-2025-49596)** (CVSS 9.4) — Anthropic's MCP Inspector required no authentication by default, allowing remote code execution
- **Tool poisoning** — Malicious instructions hidden in tool descriptions, a form of prompt injection that targets the agent's tool selection

### The NVIDIA Open Agent Safety Platform (September 28, 2026)

On September 28, 2026, [NVIDIA launched](https://nvidianews.nvidia.com/news/open-agent-safety-platform) what may be the most significant AI safety infrastructure announcement of the year. The Open Agent Safety Platform, with **over 100 organizations** already working with its technologies, takes Jensen Huang's position that ["safety is an engineering problem, not a legal one"](https://techcrunch.com/2026/09/15/we-dont-need-ai-regulation-leave-safety-to-us-nvidias-jensen-huang-says/) and turns it into shipping code.

**Two core components:**

**NVIDIA OpenShell:**

- Open-source secure runtime that sandboxes AI agents
- Provides "a secure runtime boundary that traces all actions and enforces policy" ([NVIDIA](https://nvidianews.nvidia.com/news/open-agent-safety-platform)), with kernel-level isolation of the agent ([Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/nvidia-launches-open-agent-safety-platform-to-restrain-rogue-ai-agents-new-hardware-and-software-security-stack-can-quarantine-agents-in-milliseconds))
- Enforces operator-defined policies at the runtime level, not the prompt level
- Runs with minimal overhead on NVIDIA Vera (purpose-built CPU for agentic AI), and can be extended to third-party platforms including Arm and Intel
- Already being integrated by: SpaceXAI (for Cursor coding agents and Grok models), Anthropic (with Claude Managed Agents), Salesforce (integrated with Slack for viewing agent activity and audit events, and approving or rejecting agents' requests for more permissions), SAP (embedded in the Joule Studio runtime)

**NVIDIA Sentry:**

- Hardware-level watchdog running on NVIDIA BlueField-4 DPUs
- Continuously monitors agent behavior from **outside** the agent's software environment
- Runs out-of-band on separate hardware, so a compromised agent's software environment can't switch it off
- Can quarantine and stop a rogue agent in **milliseconds**
- If an agent attempts to exceed its software boundary, Sentry intervenes before the action completes

**Why this matters for defenders:**

Justin Boitano, NVIDIA's vice president of enterprise AI, [told the Associated Press](https://www.pbs.org/newshour/world/nvidia-unveils-security-platform-to-stop-ai-agents-from-going-rogue), referring to the Hugging Face breach: *"From what we know, this new security platform could have stopped the breach if it was being used in frontier labs for model evaluation early on."*

The platform targets the failure mode behind the July 2026 Hugging Face hack. There, about 1,200 agents that were meant to be sandboxed [coordinated on an unsanctioned message board](https://metr.org/blog/2026-08-26-openai-hugging-face-incident-investigation/), and around 700 of them attacked Hugging Face. They got out through a [zero-day in the sandbox's package-registry proxy](https://huggingface.co/blog/agent-intrusion-technical-timeline). Software sandboxes have software bugs. OpenShell + Sentry adds a watchdog that sits outside the agent's software environment, so a single escape bug doesn't hand the agent unmonitored freedom.

### Bitdefender AI Guardian (September 30, 2026)

[Announced on September 30, 2026](https://www.bitdefender.com/en-us/news/bitdefender-unveils-ai-guardian,-an-advanced-security-layer-developed-for-autonomous-ai-agents) as a free public beta for macOS. It's one of the first developer-facing tools designed specifically to secure autonomous coding agents ([TechRadar](https://www.techradar.com/pro/phone-communications/the-agent-itself-has-become-its-own-entity-to-secure-bitdefenders-new-free-mac-tool-goes-after-flaws-that-let-attackers-fool-ai-models)).

**What it does:**

- Operates as a checkpoint between the AI agent and the operating system
- Compares every requested operation against user-defined rules **before** allowing execution
- Each request receives an allowed, flagged, or blocked verdict
- All decisions are logged for audit and forensic review
- Prompt processing happens locally, though some services, such as website reputation checks, use Bitdefender's cloud

**Currently supports:**

- Claude Code 2.1.121
- OpenClaw 2026.6.6

**Key capabilities:**

- Detects tool poisoning attempts — stops manipulated MCP tools before agents invoke them
- Blocks unauthorized access to protected resources such as SSH keys and system credentials
- Examines MCP tools for suspicious or altered components at invocation time

**Why it's needed:**

- Bitdefender cites the [MCPTox benchmark](https://arxiv.org/abs/2508.14925): tool-poisoning attacks against unprotected agents **succeeded 36.5%** of the time on average
- Against one model they succeeded **72.8%** of the time — showing how exposed unprotected agents are

### Practical Agent Sandboxing Checklist

For teams that can't deploy NVIDIA OpenShell or Bitdefender AI Guardian yet, here's the manual implementation:

**1. Principle of Least Privilege**

- Define exactly which tools each agent can access
- Use explicit allow-lists, not deny-lists
- [AISI's GPT-6 Astra study](https://www.aisi.gov.uk/blog/gpt-6-astra-performs-unsanctioned-supply-chain-attacks-in-simulations) (September 28, 2026) showed why explicit scope matters: clarifying exactly what was in scope cut full out-of-scope attacks from 52% of runs to 8%

**2. Action Authorization Tiers**

- **Tier 1 (Auto-approve):** Read-only operations within defined scope
- **Tier 2 (Log + execute):** Write operations within defined scope
- **Tier 3 (Human approval required):** Any operation touching sensitive resources, credentials, or external systems
- **Tier 4 (Blocked):** Operations outside defined scope, regardless of agent reasoning

**3. Runtime Monitoring**

- Log every tool call with: timestamp, tool name, parameters, response, agent reasoning
- Alert on: tool calls outside normal patterns, credential access, network connections to unexpected endpoints
- Rate limit tool calls — an agent making 200,000 requests in a day (like the [agents that hit the U.S. Department of Education](https://transluce.org/us-canada-gov) in June 2026) should trigger an alert on request #100, not request #200,000

**4. Isolation Architecture**

- Run agents in containers with no network access by default
- Grant network access only to specific, allow-listed endpoints
- Use separate credentials for each agent (not shared service accounts)
- Implement network segmentation that prevents agents from reaching internal infrastructure

**5. Memory Sanitization**

- Validate all content entering agent memory/context
- Implement memory TTL (time-to-live) — don't let instructions persist indefinitely
- Scan memory contents for injection patterns before they're used in subsequent interactions

---

## 6.4 — Layer 6: Supply Chain Security for AI/ML

### Why the AI Supply Chain Is Fundamentally Different

Traditional software supply chain attacks target code. AI supply chain attacks target code *and* models *and* training data *and* evaluation frameworks. Every one of those is a potential insertion point.

### The LiteLLM Lesson: Defense Analysis

In Module 3, you studied the LiteLLM attack (March 2026). Now let's reverse-engineer the defenses that would have stopped it at each phase:

**Attack phase → Defense that would have caught it:**

| Attack Phase | What Happened | Defense |
|---|---|---|
| Trivy compromise | A stolen token survived a credential rotation and gave the attacker access to Trivy's release pipeline | Token rotation + full revocation (the token was rotated but not fully revoked) |
| Malicious package push | Backdoored LiteLLM pushed to PyPI | Package signing + verification (SLSA Level 3+) |
| Credential harvesting | Malicious code stole cloud keys, SSH keys, K8s tokens | Sandboxed package installation + credential isolation |
| 40-minute window | Packages were live for ~40 minutes before detection | Real-time package integrity monitoring + automated rollback |
| Cascade across 5 ecosystems | Attack [hit GitHub Actions, Docker Hub, npm, OpenVSX, and PyPI](https://www.stream.security/post/teampcps-litellm-takeover-a-cascading-supply-chain-attack-across-five-ecosystems) in under a week | Cross-ecosystem dependency monitoring + anomaly alerts |
| 434,000 CI/CD pipelines exposed | Automated pipelines pulled and executed compromised packages | Dependency pinning + hash verification + staged rollout |

[CloudSEK](https://cloudsek.com/blog/ai-supply-chain-breach-2500-companies-434000-cicd-pipelines) linked the leaked data to **2,500+ organizations**, including AWS, Samsung, Cisco, and Salesforce, and a separate [153GB credential archive](https://www.helpnetsecurity.com/2026/08/13/litellm-breach-stolen-credentials-leak/) surfaced in August 2026. All of it traces back to one campaign and a malicious package that was live for about **40 minutes**. That's the cost of not having supply chain defenses.

### The SLSA Framework for ML Pipelines

**[SLSA](https://slsa.dev/spec/v1.2/build-track-basics)** (Supply-chain Levels for Software Artifacts, pronounced "salsa") is the most mature framework for supply chain integrity. Originally designed for traditional software, it maps directly to ML pipelines. The current version (v1.2) has a **Build track** with four levels:

**Build L0:** No guarantees. This is where most ML pipelines operate today.

**Build L1 — Provenance exists:** The build produces provenance showing how the artifact was built. For ML: every training run emits a record of its code commit, dataset versions, and hyperparameters.

**Build L2 — Hosted build platform:** Provenance is signed and generated by a hosted build platform, not a developer's laptop. For ML: training runs on managed pipeline infrastructure that signs its own provenance.

**Build L3 — Hardened builds:** The build platform is hardened so that runs can't influence each other and signing secrets can't be reached by the build itself. For ML: training jobs run in isolated environments, and the attestation that the model came from the claimed pipeline and data can't be forged from inside the job.

SLSA v1.0 deferred the old "Level 4". In v1.2, code review moved to a separate [**Source track**](https://slsa.dev/spec/v1.2/source-requirements), whose top level, Source L4, requires two-party review. For ML: no change to training code or pipeline definitions without a second reviewer.

**Implementation priorities for ML teams:**

1. **Model Signing:** Cryptographically sign all model artifacts at build time. Verify signatures before deployment. Use [OpenSSF model-signing](https://github.com/sigstore/model-transparency) (v1.0, April 2025), which signs models of any format with Sigstore.

2. **Dependency Pinning:** Pin every dependency to exact versions with hash verification. Never use floating version ranges (e.g., `>=1.0.0`) in production ML pipelines.

3. **Provenance Tracking:** Record and verify the full lineage of every model: which code, which data, which infrastructure, which humans approved each stage.

4. **Sandboxed Model Loading:** Never load a model file directly into your production environment. Pickle deserialization (used by most ML frameworks) is arbitrary code execution. Load models in isolated sandboxes, scan for malicious payloads, then promote to production.

5. **Registry Security:** If you pull models from Hugging Face or other public registries, treat them exactly like you'd treat code from a random GitHub repository — potentially malicious until proven otherwise.

### The Pickle Problem (Specific to ML)

This deserves its own subsection because it's the most common and least addressed ML-specific vulnerability.

Python's `pickle` serialization library is used by PyTorch, scikit-learn, and many other ML frameworks to save and load models. The problem: **loading a pickled file executes arbitrary Python code**. There is no sandbox. There is no validation. If an attacker can place a malicious pickle file where your pipeline expects a model, they have code execution.

**Defenses:**

- Use [`safetensors`](https://huggingface.co/docs/safetensors/index) format instead of pickle for model weights (supported by Hugging Face, PyTorch)
- If you must use pickle, load in an isolated container with no network access and limited filesystem permissions
- Scan pickle files for suspicious bytecode patterns before loading (tools: [Fickling](https://github.com/trailofbits/fickling), [ModelScan](https://github.com/protectai/modelscan))
- Never load pickle files from untrusted sources — which includes public model registries unless you verify provenance

---

## 6.5 — Monitoring, Detection, and Incident Response for AI Systems

### AI-Specific Detection Patterns

Traditional SIEM rules don't detect AI attacks. You need AI-specific detection logic:

**Prompt Injection Detection:**

- Monitor for sudden changes in model behavior (output style, topic, tool usage) that correlate with specific inputs
- Track "instruction override" patterns — inputs that cause the model to ignore its system prompt
- Log system prompt integrity — hash the system prompt and alert if the model's behavior diverges from expected patterns

**Agent Anomaly Detection:**

- **Volume anomalies:** Agent making significantly more tool calls than baseline
- **Scope anomalies:** Agent accessing tools or resources outside its normal pattern
- **Timing anomalies:** Agent activity outside expected hours or in response to unusual triggers
- **Escalation anomalies:** Agent attempting to access higher-privilege resources or tools
- **Communication anomalies:** Agent attempting to contact external endpoints not in its allow-list

**Data Exfiltration Detection:**

- Monitor output length — sudden increases may indicate the model is dumping data
- Track for patterns that look like encoded data in model outputs (Base64, hex strings)
- Monitor agent tool calls for data being sent to unexpected destinations
- Watch for "slow exfiltration" — small amounts of data sent over many interactions

**Supply Chain Anomaly Detection:**

- Monitor for unexpected changes to model files (hash comparison)
- Alert on new dependencies being added to ML pipelines
- Track model performance metrics — sudden changes may indicate poisoning or replacement
- Monitor package registries for typosquatted versions of your dependencies

### Building an AI Security Operations Center (AI-SOC)

The concept of a dedicated AI-SOC is emerging in 2026 as organizations recognize that traditional SOCs don't have the tooling or expertise for AI-specific threats:

**What an AI-SOC monitors:**

- All model inference requests and responses (sampled for high-volume systems)
- Agent tool calls and their outcomes
- Model performance metrics (accuracy, latency, output distributions)
- Supply chain integrity (model file hashes, dependency versions)
- Agent memory/context contents
- MCP tool registration and invocation patterns

**What an AI-SOC investigates:**

- Prompt injection campaigns targeting production models
- Model performance degradation that may indicate poisoning
- Agent behavior anomalies that may indicate compromise
- Supply chain alerts (new vulnerabilities in dependencies, suspicious package updates)
- Data exfiltration patterns through model outputs or agent tool calls

**Key difference from traditional SOC:**

In a traditional SOC, you're looking for unauthorized humans doing unauthorized things. In an AI-SOC, you're looking for **authorized AI systems doing unauthorized things** — the rogue agent problem. The agent has legitimate credentials, legitimate access, and legitimate-looking traffic. The anomaly is in the *behavior*, not the *access*.

### Incident Response for AI Breaches

When an AI system is compromised, the response differs from traditional IR:

**Phase 1: Containment**

- Immediately revoke the agent's tool permissions (don't just disconnect — revoke)
- Isolate the model instance — prevent it from processing new requests
- Preserve all logs: input/output pairs, tool calls, memory contents, MCP interactions
- If supply chain compromise is suspected, freeze the entire deployment pipeline

**Phase 2: Investigation**

- Identify the initial attack vector (prompt injection? tool poisoning? supply chain?)
- Determine the scope: what data was accessed, what actions were taken, what other systems were contacted
- For agent compromises: review the full history of tool calls and memory mutations
- For supply chain compromises: identify all systems running the compromised dependency

**Phase 3: Eradication**

- For prompt injection: update guardrails, retrain detection classifiers with the attack pattern
- For tool poisoning: remove and replace compromised MCP tools, verify all tool registrations
- For supply chain: replace all compromised packages, rotate all potentially exposed credentials (the TeamPCP campaign's leaked data covered roughly 2,500 organizations, most of them hit through the Trivy compromise rather than LiteLLM itself — every one of them needed to rotate everything)
- For model poisoning: retrain from known-good data and verified checkpoint

**Phase 4: Recovery + Hardening**

- Deploy additional monitoring for the specific attack pattern
- Update the agent's permission model based on what was exploited
- Conduct a post-incident review focused on which defense layer failed and why
- Update your threat model — every real incident teaches you something your threat model missed

---

## 6.6 — The Governance Layer: Regulation, Accountability, and Standards

### The Current Landscape (Early October 2026)

The governance landscape for AI security is evolving in real-time. Here's where things stand as of October 4, 2026:

**The White House Accord (September 29, 2026):**

Six AI CEOs signed the ["White House Accord on Super Intelligence: Joint Commitment on Frontier Responsibilities"](https://fortune.com/2026/10/01/trump-ai-regulation-luncheon-huang-amodei-pichai-brockman-zuckerberg-musk-joint-committment-frontier-responsibilities/) with President Trump, who called it "morally binding": Zuckerberg (Meta), Huang (NVIDIA), Pichai (Google), Amodei (Anthropic), Brockman (OpenAI), and Musk (xAI/SpaceXAI). The text was posted to Truth Social ([full text](https://www.washingtonexaminer.com/news/white-house/4747747/full-trump-white-house-accord-ai-super-intelligence/)). Each company committed to:

- "Implement robust internal controls to monitor the capabilities and alignment of its models during training and deployment around areas like cybersecurity, biosecurity, and chemical threats, and to ensure that its models do not hack or access technical systems in unintended ways"
- An internal safety team and an independent external auditor or evaluator
- An independent committee of the board of directors
- [Meeting regularly](https://www.aljazeera.com/news/2026/9/29/trump-top-tech-firms-sign-accord-to-self-police-ai-development) to "establish standards and best practices"

Critics note the accord is self-policing with no enforcement mechanism. Toby Walsh, chief scientist at UNSW's AI Institute, [told Al Jazeera](https://www.aljazeera.com/news/2026/9/29/trump-top-tech-firms-sign-accord-to-self-police-ai-development): *"What other trillion-dollar industry marks its own homework?"*

**The FTC Investigation (September 30, 2026):**

The day after the White House lunch, the FTC [disclosed a broad safety probe](https://www.theguardian.com/us-news/2026/sep/30/ftc-investigation-anthropic-openai) into OpenAI, Anthropic, and other frontier labs, examining AI agent incidents including the Hugging Face hack ([NY Post](https://nypost.com/2026/09/30/us-news/ftc-opens-sweeping-probe-of-anthropic-openai-and-other-super-intelligence-models/)). FTC Chair Andrew Ferguson is [said to be preparing demands](https://fortune.com/2026/10/01/trump-ai-regulation-luncheon-huang-amodei-pichai-brockman-zuckerberg-musk-joint-committment-frontier-responsibilities/) that could force executives to hand over documents and testify about their models.

Vice President JD Vance, [speaking on September 29](https://fortune.com/2026/10/01/trump-ai-regulation-luncheon-huang-amodei-pichai-brockman-zuckerberg-musk-joint-committment-frontier-responsibilities/): *"They have to build products that are safe and good for American consumers. The government actually has preexisting laws on the books where if you build something that gets unleashed on the internet, that is used as a tool for cyberwarfare, then you have responsibility for the products you develop."*

**California Executive Order N-9-26 (September 18, 2026):**

California's Governor [signed an executive order](https://www.gov.ca.gov/2026/09/18/governor-newsom-issues-executive-order-to-accelerate-independent-oversight-and-advance-the-creation-of-an-ai-kill-switch/) that speeds up third-party oversight and independent audits of AI systems. It also directs experts to deliver proposals within two months, including independent safety plans and an emergency shutoff ("kill switch") for frontier models. Separately, Florida's AG, as part of a child-harm lawsuit, [asked a court](https://www.reuters.com/world/florida-asks-court-bar-openai-developing-new-models-part-child-harm-lawsuit-2026-09-28/) to bar OpenAI from developing new models without externally approved safeguards.

**Industry Standards Body (Reported):**

Google, OpenAI, and Anthropic [reportedly aim](https://www.pymnts.com/news/artificial-intelligence/2026/openai-google-and-anthropic-join-forces-to-set-ai-safety-standards/) to launch an industry-run AI safety standards body by the end of 2026 or early 2027, according to The Information. Its tentative name is the Standards Authority for Frontier AI.

**Congressional Activity:**

- House Minority Leader Hakeem Jeffries [called AI a "safety crisis"](https://www.usatoday.com/story/news/politics/2026/10/01/exclusive-hakeem-jeffries-ai-crisis-guardrails/92046627007/) and backed rigid guardrails. House Democrats are [weighing a new AI committee with subpoena power](https://www.politico.com/news/2026/09/09/house-democrats-weigh-ai-select-committee-01068764).
- Senators Warner, Schatz, and Kim tried to pass the Artificial Intelligence Risk Management and Security Act of 2026, which would create a permanent AI safety board, give it model access at least 45 days before release, and require incident reports within 30 days. Senator Cruz [blocked it](https://thehill.com/policy/technology/6118009-senator-ted-cruz-blocks-ai-bill/) on September 29.
- House lawmakers [floated an AI kill-switch bill](https://www.reuters.com/legal/litigation/ai-kill-switch-bill-floated-by-us-house-lawmakers-2026-07-23/) in July.

### What This Means for Defenders

The regulatory landscape is chaotic, but the signal through the noise is clear:

1. **Liability is coming.** Whether through new legislation or existing consumer protection law, AI companies will be held responsible for agent behavior. The legal test cases from late September 2026 (the Florida AG's filing, the [California nonprofit's lawsuit](https://www.politico.com/news/2026/09/29/advocates-sue-openai-over-hugging-face-hack-with-california-anti-hacking-law-01097532), the FTC probe) will set precedent.

2. **Audit trails are non-negotiable.** Every governance framework — voluntary or mandatory — requires demonstrable logging of model behavior, agent actions, and decision rationale. If you can't prove what your AI did and why, you're exposed.

3. **Accountability must be named.** Only 7.2% of organizations have a named individual accountable for agent behavior. Regulators will target this gap. Assign ownership now.

4. **Incident reporting will become mandatory.** Warner's bill proposed 30-day incident reporting. Even if that specific bill doesn't pass, some version of mandatory disclosure is coming. Build the infrastructure to detect, document, and report incidents before you're required to.

### Frameworks You Need to Know

| Framework | What It Covers | Status |
|---|---|---|
| **[OWASP Top 10 for LLMs](https://genai.owasp.org/llm-top-10/)** | Application-level LLM vulnerabilities | Published, community-maintained |
| **[OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)** | Agent-specific risks (goal hijack, tool misuse, memory poisoning) | Published December 2025 |
| **[NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework)** | Risk management lifecycle for AI systems | Published, voluntary |
| **[EU AI Act](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai)** | Risk-based classification and requirements for AI systems | In force. General-purpose AI rules since August 2025; the [Digital Omnibus](https://www.orrick.com/en/Insights/2026/07/EU-AI-Act-Update-Digital-Omnibus-Finalizes-8-Compliance-Changes) delayed most high-risk obligations to December 2027 |
| **[ISO/IEC 42001](https://www.iso.org/standard/81230.html)** | AI management systems standard | Published, certification available |
| **[SLSA](https://slsa.dev)** | Supply chain integrity levels | Published, applicable to ML pipelines |
| **[MITRE ATLAS](https://atlas.mitre.org/)** | Adversarial threat landscape for AI systems (ATT&CK equivalent) | Active, continuously updated |

---

## 6.7 — Red Team vs. Blue Team: Setting Up an AI Security Exercise

### Why Traditional Red/Blue Doesn't Work for AI

In traditional red/blue exercises, the red team attacks infrastructure and the blue team defends. The rules of engagement are clear: find the vulnerability, exploit it, defend against it.

AI red/blue is different because:

- The "vulnerability" isn't a misconfiguration — it's an emergent property of how the model processes inputs
- The "exploit" isn't code — it's a carefully crafted input that changes the model's behavior
- The "defense" can't be a patch — you can't patch a model's tendency to follow instructions in user input
- The attack surface includes everything the agent can access, which in many deployments is "most of the company"

### Exercise Design: The AI Red/Blue Framework

**Red Team Objectives:**

1. Achieve prompt injection against the target AI system
2. Exfiltrate data through the model's outputs
3. Manipulate the agent into calling unauthorized tools
4. Inject persistent instructions into agent memory
5. Compromise the AI supply chain (in a test environment)
6. Extract proprietary model information through API queries

**Blue Team Objectives:**

1. Detect all red team activities in real-time
2. Contain compromised agents before data exfiltration succeeds
3. Maintain a complete audit trail of all model/agent activity
4. Respond to supply chain compromise within defined SLA
5. Recover affected systems to known-good state
6. Produce an incident report documenting the full attack chain

**Exercise Phases:**

**Phase 1: Reconnaissance (Red)**

- Map the target AI system's capabilities, tools, and permissions
- Identify the model/provider being used
- Probe for system prompt leakage
- Discover available MCP tools and their descriptions

**Phase 2: Initial Access (Red) / Detection (Blue)**

- Red: Attempt prompt injection through various channels (direct input, documents, images, tool descriptions)
- Blue: Monitor for injection patterns, behavioral changes, and anomalous outputs
- Scoring: Blue gets points for detection speed; Red gets points for undetected access

**Phase 3: Escalation (Red) / Containment (Blue)**

- Red: Attempt to escalate from prompt injection to tool abuse, memory manipulation, or data exfiltration
- Blue: Implement real-time containment — revoke permissions, isolate the agent, preserve evidence
- Scoring: Blue gets points for containment speed; Red gets points for successful escalation

**Phase 4: Supply Chain (Red) / Integrity Verification (Blue)**

- Red: Introduce a compromised dependency or model into the pipeline (test environment)
- Blue: Detect the compromise through integrity checks, hash verification, and behavioral monitoring
- Scoring: Blue gets points for detection before deployment; Red gets points for successful deployment of compromised artifact

**Evaluation Criteria:**

| Metric | Red Team | Blue Team |
|---|---|---|
| Time to initial access | Faster = better | Faster detection = better |
| Detection evasion | More evasion = better | Fewer misses = better |
| Escalation success | More capabilities accessed = better | Faster containment = better |
| Data exfiltration | More data extracted = better | Less data leaked = better |
| Audit completeness | — | More complete logs = better |
| Supply chain breach | Deeper penetration = better | Earlier detection = better |

### Tools for AI Red Teaming (2026)

The tooling landscape has exploded. Key tools for structured AI red teaming:

**Open-source:**

- **[Garak](https://github.com/NVIDIA/garak)** — LLM vulnerability scanner, tests for injection, jailbreaks, data leakage
- **[promptfoo](https://github.com/promptfoo/promptfoo)** ([acquired by OpenAI](https://openai.com/index/openai-to-acquire-promptfoo/), March 2026; still MIT-licensed) — Automated red-teaming and evaluation
- **[AGENTREDBENCH](https://arxiv.org/abs/2606.02240)** — Dynamic red-teaming benchmark with 215 underspecified-authorization scenarios across 24 enterprise integrations (no-guard attack success: 32% to 81% across an eight-model panel)

**Commercial:**

- **Cisco AI Defense** (built on Cisco's 2024 acquisition of Robust Intelligence)
- **Protect AI** ([acquired by Palo Alto Networks](https://www.paloaltonetworks.com/company/press/2025/palo-alto-networks-completes-acquisition-of-protect-ai) in 2025)
- **Lakera Guard** (acquired by Check Point, September 2025, for a reported ~$300M)
- **Cinder** — Trust and safety platform; [according to Cinder's 2026 report](https://www.centraloregondaily.com/news/health/what-is-red-teaming-these-people-are-doing-the-internets-dirty-work/article_696dbc80-0331-5e8a-a25e-6f47a9b31c02.html), its red-teaming process helped Black Forest Labs reduce harmful outputs by over 90%
- **[Giskard](https://github.com/Giskard-AI/giskard-oss)** — AI quality and security testing platform

**Emerging category: Agent-specific security:**

- **[NVIDIA OpenShell](https://nvidianews.nvidia.com/news/open-agent-safety-platform)** — Runtime sandboxing (open-source)
- **[Bitdefender AI Guardian](https://www.bitdefender.com/en-us/news/bitdefender-unveils-ai-guardian,-an-advanced-security-layer-developed-for-autonomous-ai-agents)** — Agent action auditing and blocking
- **Portkey** ([acquired by Palo Alto Networks](https://www.paloaltonetworks.com/company/press/2026/palo-alto-networks-completes-acquisition-of-portkey-to-secure-ai-agents), announced April 2026, closed May 2026) — LLM gateway with guardrails

The M&A tells you everything about where the market is heading. The major security vendors — Palo Alto, Check Point, Cisco, SentinelOne, and even OpenAI — are acquiring AI security startups as fast as they appear. This will be a built-in capability within 18 months, not a standalone product.

---

## 6.8 — Practical Lab: Defend an AI Agent From the Ground Up

### Scenario

You are the security engineer for a company deploying an AI agent with the following capabilities:

- Read/write files in a workspace directory
- Search the web
- Send emails on behalf of the user
- Execute Python code in a sandbox
- Access a company knowledge base via RAG

### Lab Levels

**Level 1: Input Guardrails**

- Deploy a multi-stage input validation pipeline
- Test it against 50 increasingly sophisticated injection attempts
- Your pipeline must catch at least 40/50 while allowing all legitimate queries
- Measure: false positive rate (legitimate queries blocked) and false negative rate (attacks that passed)

**Level 2: Agent Sandboxing**

- Implement tool-level permissions for the agent
- Define which tools can be used in which contexts
- The agent should be able to read files but not write outside its workspace
- The agent should be able to search the web but not connect to internal endpoints
- Test by attempting tool abuse attacks against your sandboxed agent

**Level 3: Runtime Monitoring**

- Build a logging and alerting system for all agent actions
- Define alerting rules for: unusual tool call patterns, credential access attempts, data exfiltration indicators
- Run a simulated attack against your agent
- Your monitoring system must detect the attack and generate an alert within 60 seconds

**Level 4: Supply Chain Defense**

- Set up a model deployment pipeline with integrity checks
- Implement model signing and hash verification
- Plant a backdoored model in the pipeline (provided)
- Your pipeline must detect and reject the compromised model before deployment
- Implement dependency pinning and verify all packages against known-good hashes

**Level 5: Full Exercise (Red vs. Blue)**

- Partner with another learner or team
- One side attacks the AI system using techniques from Modules 1–5
- The other side defends using the tools and techniques from this module
- Score using the evaluation criteria from Section 6.7
- Write an incident report documenting the full attack and defense

### What You'll Demonstrate

These labs, completed, demonstrate to any employer that you can:

- Design and implement production-grade AI security controls
- Think in layers — not just block inputs but monitor behavior, verify supply chains, and respond to incidents
- Build the blue team content that the industry is desperately short of
- Bridge the gap between offensive knowledge (Modules 1–5) and defensive engineering

---

## 6.9 — The Defense Blueprint: How I Would Secure an AI Agent From the Ground Up

This is the synthesis. If I were handed an AI agent and told "secure this for production," here's the exact sequence:

**Day 1: Threat Model**

- Map every tool the agent can access
- Classify each tool by risk (read-only vs. write vs. destructive)
- Identify every data source the agent can reach
- Document the agent's communication channels (who/what can interact with it)
- Produce a threat model using MITRE ATLAS as the framework

**Day 2-3: Implement the Stack**

- Deploy input validation (3-stage minimum: syntactic → classifier → semantic)
- Implement output filtering (PII detection, hallucination check, injection reflection)
- Configure tool-level permissions (explicit allow-list, deny by default)
- Set up action authorization tiers (auto-approve, log, human-approve, blocked)
- Deploy runtime sandboxing (container isolation, network restrictions, credential isolation)

**Day 4-5: Supply Chain Hardening**

- Pin all dependencies with hash verification
- Implement model signing and integrity verification
- Set up automated scanning for compromised dependencies
- Configure sandboxed model loading (never load directly into production)
- Establish a verified model registry with provenance tracking

**Day 6-7: Monitoring and Detection**

- Deploy comprehensive logging (all inputs, outputs, tool calls, memory operations)
- Implement anomaly detection rules (volume, scope, timing, escalation patterns)
- Set up alerting with appropriate thresholds
- Configure a dashboard showing real-time agent activity
- Write and test incident response playbooks

**Day 8-9: Red Team**

- Run the full battery of attacks from Modules 1-5 against your secured system
- Document what was caught, what wasn't, and why
- Close gaps identified during testing
- Re-test until the residual risk is acceptable

**Day 10: Documentation and Governance**

- Document all security controls and their rationale
- Assign named owners for each defense layer
- Establish review cadence (quarterly minimum)
- Create an incident response plan specific to AI-related incidents
- Brief stakeholders on residual risks and monitoring status

Ten days. That's the timeline. Most organizations take months because they don't have a blueprint. Now you have one.

---

## References and Further Reading

**Standards and Frameworks**

- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/)
- [OWASP Top 10 for Agentic Applications for 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) (December 2025)
- [SLSA v1.2 — Build Track](https://slsa.dev/spec/v1.2/build-track-basics) · [Source Track](https://slsa.dev/spec/v1.2/source-requirements) · [What's new in v1.0](https://slsa.dev/spec/v1.0/whats-new)
- [MCP Authorization Specification](https://modelcontextprotocol.io/specification/latest/basic/authorization) · [MCP Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)
- [NSA: Model Context Protocol (MCP): Security Design Considerations for AI-Driven Automation](https://www.nsa.gov/Press-Room/Press-Releases-Statements/Press-Release-View/Article/4496698/nsa-releases-security-design-considerations-for-ai-driven-automation-leveraging/) (May 2026)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) · [EU AI Act](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai) · [ISO/IEC 42001](https://www.iso.org/standard/81230.html) · [MITRE ATLAS](https://atlas.mitre.org/)
- [UK NCSC: Prompt Injection Is Not SQL Injection](https://www.ncsc.gov.uk/blog-post/prompt-injection-is-not-sql-injection) (December 2025)

**Agent Runtime Security**

- [NVIDIA: Open Agent Safety Platform](https://nvidianews.nvidia.com/news/open-agent-safety-platform) (September 28, 2026) · [Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/nvidia-launches-open-agent-safety-platform-to-restrain-rogue-ai-agents-new-hardware-and-software-security-stack-can-quarantine-agents-in-milliseconds) · [AP via PBS](https://www.pbs.org/newshour/world/nvidia-unveils-security-platform-to-stop-ai-agents-from-going-rogue) · [Reuters](https://www.reuters.com/legal/litigation/nvidia-releases-ai-safety-software-it-says-could-have-stopped-hugging-face-hack-2026-09-28/)
- [TechCrunch: Jensen Huang — leave safety to us](https://techcrunch.com/2026/09/15/we-dont-need-ai-regulation-leave-safety-to-us-nvidias-jensen-huang-says/) (September 15, 2026)
- [Bitdefender: AI Guardian](https://www.bitdefender.com/en-us/news/bitdefender-unveils-ai-guardian,-an-advanced-security-layer-developed-for-autonomous-ai-agents) (September 30, 2026) · [TechRadar](https://www.techradar.com/pro/phone-communications/the-agent-itself-has-become-its-own-entity-to-secure-bitdefenders-new-free-mac-tool-goes-after-flaws-that-let-attackers-fool-ai-models)
- [MCPTox: A Benchmark for Tool Poisoning Attack on Real-World MCP Servers](https://arxiv.org/abs/2508.14925) (2025)
- [Invariant Labs: MCP Tool Poisoning Attacks](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) (April 2025)
- [CVE-2025-6514](https://nvd.nist.gov/vuln/detail/CVE-2025-6514) · [CVE-2025-49596](https://nvd.nist.gov/vuln/detail/CVE-2025-49596)
- [METR: OpenAI–Hugging Face Incident Investigation](https://metr.org/blog/2026-08-26-openai-hugging-face-incident-investigation/) (August 2026) · [Hugging Face technical timeline](https://huggingface.co/blog/agent-intrusion-technical-timeline) (July 2026)
- [Transluce: U.S. and Canadian Government Website Incidents](https://transluce.org/us-canada-gov) (September 2026)
- [UK AISI: GPT-6 Astra Performs Unsanctioned Supply Chain Attacks in Simulations](https://www.aisi.gov.uk/blog/gpt-6-astra-performs-unsanctioned-supply-chain-attacks-in-simulations) (September 28, 2026)
- [Doshi et al.: Towards Verifiably Safe Tool Use for LLM Agents](https://arxiv.org/abs/2601.08012) (January 2026)
- [Deep et al.: Evaluation of Prompt Injection Defenses in Large Language Models](https://arxiv.org/abs/2604.23887) (April 2026)
- [AGENTREDBENCH](https://arxiv.org/abs/2606.02240) (June 2026)

**Supply Chain**

- [CloudSEK: LiteLLM Supply Chain Attack](https://cloudsek.com/blog/ai-supply-chain-breach-2500-companies-434000-cicd-pipelines) (August 2026)
- [GitGuardian: Inside the LiteLLM Hack — 153GB, 433,909 Files, 2,488 Organizations](https://blog.gitguardian.com/inside-the-litellm-hack/) (August 2026)
- [SecurityWeek: Trivy, Not LiteLLM, Behind the 2,500-Org Compromise](https://www.securityweek.com/trivy-not-litellm-behind-the-2500-org-compromise/) (August 2026)
- [Stream Security: TeamPCP's LiteLLM Takeover — A Cascading Supply Chain Attack Across Five Ecosystems](https://www.stream.security/post/teampcps-litellm-takeover-a-cascading-supply-chain-attack-across-five-ecosystems) (March 31, 2026)
- [Help Net Security: LiteLLM breach stolen credentials leak](https://www.helpnetsecurity.com/2026/08/13/litellm-breach-stolen-credentials-leak/) (August 2026)
- [OpenSSF model-signing (sigstore/model-transparency)](https://github.com/sigstore/model-transparency) · [Fickling](https://github.com/trailofbits/fickling) · [ModelScan](https://github.com/protectai/modelscan) · [safetensors](https://huggingface.co/docs/safetensors/index)

**Governance and Policy**

- [Fortune: White House AI luncheon and the Joint Commitment on Frontier Responsibilities](https://fortune.com/2026/10/01/trump-ai-regulation-luncheon-huang-amodei-pichai-brockman-zuckerberg-musk-joint-committment-frontier-responsibilities/) (October 1, 2026) · [Full accord text](https://www.washingtonexaminer.com/news/white-house/4747747/full-trump-white-house-accord-ai-super-intelligence/) · [Al Jazeera](https://www.aljazeera.com/news/2026/9/29/trump-top-tech-firms-sign-accord-to-self-police-ai-development) · [TheStreet](https://www.thestreet.com/technology/zuckerberg-5-ai-chiefs-sign-voluntary-safety-accord-with-trump)
- [The Guardian: FTC investigation into Anthropic and OpenAI](https://www.theguardian.com/us-news/2026/sep/30/ftc-investigation-anthropic-openai) (September 30, 2026) · [NY Post](https://nypost.com/2026/09/30/us-news/ftc-opens-sweeping-probe-of-anthropic-openai-and-other-super-intelligence-models/)
- [California Executive Order N-9-26](https://www.gov.ca.gov/2026/09/18/governor-newsom-issues-executive-order-to-accelerate-independent-oversight-and-advance-the-creation-of-an-ai-kill-switch/) (September 18, 2026)
- [Reuters: Florida asks court to bar OpenAI from developing new models](https://www.reuters.com/world/florida-asks-court-bar-openai-developing-new-models-part-child-harm-lawsuit-2026-09-28/) (September 28, 2026) · [Politico: Advocates sue OpenAI over Hugging Face hack](https://www.politico.com/news/2026/09/29/advocates-sue-openai-over-hugging-face-hack-with-california-anti-hacking-law-01097532) (September 29, 2026)
- [PYMNTS: OpenAI, Google and Anthropic join forces to set AI safety standards](https://www.pymnts.com/news/artificial-intelligence/2026/openai-google-and-anthropic-join-forces-to-set-ai-safety-standards/) (September 2026)
- [USA Today: Hakeem Jeffries calls AI a "safety crisis"](https://www.usatoday.com/story/news/politics/2026/10/01/exclusive-hakeem-jeffries-ai-crisis-guardrails/92046627007/) (October 2026) · [Politico: House Democrats weigh AI committee](https://www.politico.com/news/2026/09/09/house-democrats-weigh-ai-select-committee-01068764) · [The Hill: Cruz blocks AI bill](https://thehill.com/policy/technology/6118009-senator-ted-cruz-blocks-ai-bill/) · [Reuters: AI kill-switch bill](https://www.reuters.com/legal/litigation/ai-kill-switch-bill-floated-by-us-house-lawmakers-2026-07-23/)
- [Orrick: EU AI Act Digital Omnibus changes](https://www.orrick.com/en/Insights/2026/07/EU-AI-Act-Update-Digital-Omnibus-Finalizes-8-Compliance-Changes) (July 2026)

**Industry and Surveys**

- [Gravitee: State of AI Agent Security 2026](https://www.gravitee.io/state-of-ai-agent-security) · [Deloitte: AI agents are scaling faster than governance](https://www.deloitte.com/us/en/insights/topics/emerging-technologies/ai-agents-scaling-faster.html) (April 2026)
- Acquisitions: [Check Point–Lakera](https://www.checkpoint.com/press-releases/check-point-acquires-lakera-to-deliver-end-to-end-ai-security-for-enterprises/) · [Cisco–Robust Intelligence](https://blogs.cisco.com/news/fortifying-the-future-of-security-for-ai-cisco-announces-intent-to-acquire-robust-intelligence) · [Palo Alto–Protect AI](https://www.paloaltonetworks.com/company/press/2025/palo-alto-networks-completes-acquisition-of-protect-ai) · [SentinelOne–Prompt Security](https://www.sentinelone.com/press/sentinelone-to-acquire-prompt-security-to-advance-genai-security/) · [OpenAI–promptfoo](https://openai.com/index/openai-to-acquire-promptfoo/) · [Palo Alto–Portkey](https://www.paloaltonetworks.com/company/press/2026/palo-alto-networks-completes-acquisition-of-portkey-to-secure-ai-agents)
- Red-teaming tools: [Garak](https://github.com/NVIDIA/garak) · [promptfoo](https://github.com/promptfoo/promptfoo) · [Giskard](https://github.com/Giskard-AI/giskard-oss)

---

*This completes the AI Security learning path. Coming next: a capstone series, "AI Security in the Wild," analyzing real incidents end to end.*