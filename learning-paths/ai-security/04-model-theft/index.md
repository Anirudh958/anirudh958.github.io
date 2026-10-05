---
layout: module
title: "Model Theft — Stealing Weights, Not Just Queries"
path_id: ai-security
module: 4
description: "How adversaries steal proprietary AI models through API extraction, side-channel attacks on GPU infrastructure, training data regurgitation, and reasoning trace replay — plus the economics that make model theft the fastest-growing AI threat of 2026."
---

*By Anirudh Gupta Surisetty · Last updated October 2026 · ~15 min read · Prerequisite: Module 3 — AI Supply Chain Attacks*

## Why This Module Exists

Here's something the AI industry doesn't want to talk about openly.

Training a frontier model costs somewhere between $100 million and $1 billion. That was [Anthropic's CEO's estimate in 2024](https://www.tomshardware.com/tech-industry/artificial-intelligence/ai-models-that-cost-dollar1-billion-to-train-are-in-development-dollar100-billion-models-coming-soon-largest-current-models-take-only-dollar100-million-to-train-anthropic-ceo), and [Epoch AI](https://arxiv.org/abs/2405.21015) found training costs growing about 2.4x per year. The weights — those billions of floating-point numbers that encode everything the model learned — represent the most expensive intellectual property humanity has ever created per unit of storage.

And they're being stolen. Regularly.

Not through dramatic heists. Through carefully crafted API queries. Through electromagnetic emissions leaking from GPUs. Through getting the model itself to vomit up its own training data. And, as of September 2026, through replaying encrypted reasoning traces across sessions until the model decrypts its own thoughts for you.

Model theft isn't hypothetical. On September 30, 2026, [OpenAI disclosed](https://openai.com/index/disrupting-a-coordinated-model-distillation-campaign/) a coordinated campaign to extract hidden reasoning from its models, attributing a core cluster of the activity to individuals associated with China-based Moonshot AI. At its peak the campaign sent 16,000 extraction requests over two days from more than 4,000 users. Weeks earlier, [Anthropic reported](https://www.anthropic.com/threat-intelligence-report-september-2026) that Moonshot had relayed almost 300,000 customer requests to Anthropic over a ten-day period, through a proxy network of 5,380 fraudulent accounts. Moonshot saved Claude's reasoning signatures and replayed them in new sessions.

This is the module where we stop talking about prompts and start talking about the thing that actually keeps AI companies awake at night: someone stealing the model itself.

---

## 4.1 — The Economics of Model Theft

Before we get into techniques, understand *why* this attack class is exploding.

**The asymmetry is staggering:**

- **Cost to train a frontier model:** On the order of $100 million to $1 billion as of 2024, and rising fast
- **Cost to steal useful pieces via the API:** Tiny by comparison. [Knockoff Nets](https://arxiv.org/abs/1812.02766) built a working copy of an image classifier for as little as $30. [Carlini et al.](https://arxiv.org/abs/2403.06634) recovered the full embedding projection layer of OpenAI's Ada and Babbage models for under $20.
- **Cost to extract reasoning traces via replay attacks:** Close to free — the attacker uses the victim's own infrastructure

This is the same dynamic that drives software piracy, except the "software" cost a billion dollars to create and can be approximated through its own outputs. The economic incentive is unlike anything the security industry has seen before.

**Who's doing this:**

- **Nation-state actors** — building sovereign AI capabilities without the compute investment
- **Competitor companies** — "adversarial distillation" is the polite term for what's happening
- **Research groups** — extracting proprietary architectures to publish papers about them
- **Criminal enterprises** — stealing models to strip safety guardrails and resell access

MITRE ATLAS maps this activity under the [AI Model Access](https://atlas.mitre.org/tactics/AML.TA0000) tactic (AML.TA0000). The core techniques are [AML.T0024.002 — Extract AI Model](https://atlas.mitre.org/techniques/AML.T0024.002), [AML.T0005 — Create Proxy AI Model](https://atlas.mitre.org/techniques/AML.T0005), and [AML.T0048.004 — AI Intellectual Property Theft](https://atlas.mitre.org/techniques/AML.T0048.004). ATLAS even added a case study for the Claude distillation campaigns ([AML.CS0056](https://atlas.mitre.org/studies/AML.CS0056)). But the taxonomy is evolving faster than the framework can keep up.

---

## 4.2 — Model Extraction via API: The Query Game

### The Core Principle

Every time a model responds to a query, it leaks information about itself. The logits (output probabilities), the confidence scores, even the raw text — all of it encodes information about the model's internal decision boundaries.

Model extraction is the art of sending strategically crafted queries to reconstruct enough of those decision boundaries that you can build a functionally equivalent model — a "clone" that behaves identically on inputs that matter to the attacker.

### How It Works — The Classical Approach

**Step 1: Reconnaissance**

The attacker identifies what the target model does — classification, generation, regression — and what outputs are available. Full probability distributions? Top-k logits? Just the text?

**Step 2: Query Crafting**

This is where it gets interesting. Random queries are inefficient. Smart attackers use *active learning* strategies:

- **Boundary queries** — inputs designed to land near decision boundaries, where the model's responses are most informative
- **Uncertainty sampling** — queries where the model is least confident, revealing the most about internal structure
- **Counterfactual probing** — systematically varying inputs to map how small changes affect outputs

**Step 3: Distillation**

The attacker trains a "student" model on the query-response pairs. The student doesn't need to learn from scratch — it's learning from the teacher's *outputs*, which encode the teacher's knowledge far more efficiently than raw training data.

**Step 4: Fidelity Testing**

The attacker measures how well their clone matches the original on held-out inputs. Against simple models, extraction can reach [near-perfect fidelity](https://arxiv.org/abs/1609.02943): Tramèr et al. cloned logistic regressions, neural networks, and decision trees served by BigML and Amazon ML.

### The Moonshot-OpenAI Campaign (July 2026) — A Live Case Study

This is no longer theoretical. Here's what [OpenAI disclosed](https://openai.com/index/disrupting-a-coordinated-model-distillation-campaign/) on September 30, 2026:

**The target:** OpenAI's reasoning models, specifically their "protected reasoning" — the model's internal chain-of-thought that is encrypted and transmitted as an opaque blob to the client.

**The technique:** Cross-session reasoning trace replay. OpenAI's architecture encrypts the model's reasoning and passes it to the client, which sends it back with subsequent requests so the server doesn't have to store session state. The attackers:

1. Collected encrypted reasoning traces from one conversation
2. Injected those traces into a *different* conversation
3. Asked the model in that conversation to decrypt and transcribe the hidden reasoning in plaintext

**The scale:**

- Activity started July 1, 2026, at low volume
- Peaked on July 24–25 with **16,000 extraction requests** from **over 4,000 users**
- Related activity spanned a cluster of more than 15,000 users, and was fully disrupted by July 28
- OpenAI notes these were *attempted* extractions, not necessarily successful ones

**The parallel Anthropic attack:**

[Anthropic's September 2026 threat report](https://www.anthropic.com/threat-intelligence-report-september-2026) describes a matching campaign. Over one ten-day period, Moonshot relayed **almost 300,000 customer requests** to Anthropic through a proxy network of **5,380 fraudulent accounts**. These were real Kimi customers' requests, silently forwarded to Claude, mostly to Opus. Moonshot saved the reasoning signature from each Claude response, started a new session, and got Claude to convert the signature back into the full reasoning trace. Anthropic counted more than 23 million exchanges from Moonshot between May and July 2026.

**What this tells us:**

- The encryption wasn't broken — the *architecture* was exploited
- The attack used the model's own capabilities against itself
- "Adversarial distillation" at this scale is a coordinated intelligence operation, not a hobbyist project

Independent researchers confirmed the attack paths in their paper ["Stealing Reasoning Traces from Proprietary LLM APIs"](https://arxiv.org/abs/2608.09867) (August 2026), testing against OpenAI, Anthropic, and Google. They injected a frontier model's encrypted reasoning into a weaker, less-safeguarded model from the *same provider*, and the weaker model decoded it and wrote it out verbatim in plaintext. Decoding 315,320 public reasoning blocks surfaced 367 PII artifacts and 182 credentials.

### Defense Reality Check

After the Moonshot campaign, [OpenAI](https://openai.com/index/disrupting-a-coordinated-model-distillation-campaign/):

- Closed a pathway that let someone holding another user's encrypted reasoning replay it and recover its contents
- Banned or restricted the accounts involved
- Added checks to detect and hold streamed output that might expose reasoning
- Strengthened protections across users, workspaces, organizations, and model families
- Shared findings through the Frontier Model Forum and government information-sharing channels

But OpenAI's own disclosure says the quiet part out loud: *"Systems that support portable or replayable reasoning artifacts may face related risks."* Translation: any model architecture that externalizes reasoning state is fundamentally vulnerable to replay extraction.

---

## 4.3 — Side-Channel Attacks on GPU Infrastructure

This is where model theft meets hardware hacking. And it's far more real than most people think.

### The Attack Surface

Modern AI inference runs on shared GPU infrastructure — cloud providers colocate multiple tenants on the same physical hardware. Every computation the GPU performs leaves physical traces:

- **Electromagnetic (EM) emissions** — the GPU radiates EM signals that correlate with the operations it's performing
- **Power consumption** — different operations draw different amounts of power, creating measurable patterns
- **Timing variations** — the time a model takes to process an input leaks information about the model's architecture
- **Thermal signatures** — computation generates heat patterns that correlate with model structure
- **Cache access patterns** — shared cache hierarchies leak information about memory access patterns

### Kraken: EM Side-Channel Extraction (2026)

The ["Kraken" research](https://arxiv.org/abs/2603.02891) (March 2026, IEEE SaTML 2026) demonstrated higher-order electromagnetic side-channel attacks against DNNs running on GPU **Tensor Cores**. Earlier work had only targeted CUDA cores. Key findings:

- Using near-field Correlation Power Analysis on the GPU's EM emanations during inference, researchers extracted model parameters
- In the **far field**, they observed hyperparameter and weight leakage from LLMs **up to 100 cm away, through a glass obstacle**. You don't need physical contact with the GPU
- Tensor Cores, the hardware that makes modern AI inference fast, are not immune to physical side channels

Think about what that means: by passively observing a GPU's EM emissions from across a room, an attacker can start to recover the model running on it.

### Energon: Power and Thermal Side-Channels (2025)

The ["Energon" attack](https://arxiv.org/abs/2508.01768) (ICCAD 2025) demonstrated that transformer model architectures can be reverse-engineered from GPU power and temperature readings alone, read through `nvidia-smi` at ordinary user privilege:

- Identified the model family over 89% of the time and classified hyperparameters (encoder/decoder layers, attention heads) with 100% accuracy
- Used the recovered architecture to mount **black-box transfer adversarial attacks with over 93% success**
- Tested on NVIDIA hardware, including an A40
- The attack is *software-visible*: it works in noisy multi-process settings, the kind you get on shared GPUs and ML-as-a-service platforms, without physical access

### The Hot Pixels Lineage

Building on the ["Hot Pixels" research](https://arxiv.org/abs/2305.12784) (USENIX Security 2023), which showed that frequency, power, and temperature can be exploited as hybrid side channels on GPUs and ARM SoCs:

- Modern DVFS (Dynamic Voltage and Frequency Scaling) mechanisms create software-visible side channels
- These bypass countermeasures designed for traditional microarchitectural attacks
- In cloud environments with shared GPU infrastructure, this means *your model's architecture might be leaking to whoever is running on the adjacent GPU partition*

### Why This Matters for AI Security

Side-channel attacks on GPUs represent a **Layer 5 (Infrastructure)** threat from our attack surface model. Most AI security discussions focus on Layers 1-3. But if an attacker can extract your model architecture through EM emissions from a shared cloud GPU, all your prompt injection defenses are irrelevant — they've already got the model.

**Current defenses are limited:**

- [PermuteV](https://arxiv.org/abs/2512.18132) (December 2025) randomly permutes the execution order of loop iterations to obfuscate EM signatures. But it is a modified RISC-V core for edge AI inference, so it requires custom hardware
- Dedicated GPU instances eliminate co-tenancy attacks but dramatically increase cost
- Constant-time inference implementations exist but impose significant performance penalties

---

## 4.4 — Training Data Extraction: Making the Model Confess

### The Memorization Problem

Here's a fact that should concern every organization using LLMs: **language models memorize their training data**. Not just patterns — actual verbatim text.

[Carlini et al.'s foundational research](https://arxiv.org/abs/2012.07805) (USENIX Security 2021) demonstrated that individual training examples can be recovered by querying a language model. Subsequent work showed:

- [Over 1% of the unprompted output](https://arxiv.org/abs/2107.06499) of language models trained on common web datasets is copied verbatim from the training data. GPT-J 6B [memorizes at least 1% of its training set](https://arxiv.org/abs/2202.07646), and given enough context it reproduces a training sequence's continuation about 7% of the time
- When ChatGPT (gpt-3.5-turbo) was [prompted to repeat a single word forever](https://arxiv.org/abs/2311.17035), it eventually diverged and started emitting **raw training data**. Researchers recovered over 10,000 examples for $200
- These aren't edge cases. Memorization [grows with model size](https://arxiv.org/abs/2202.07646), with how often an example is duplicated, and with how much context the attacker supplies

### Attack Taxonomy

**Untargeted extraction** — generating large volumes of model output and identifying memorized training data through membership inference:

1. Generate thousands of outputs using diverse prompts
2. Use a membership inference classifier to identify which outputs closely match training data
3. Recover personal information, copyrighted content, or proprietary data

**Targeted extraction ([Neural Phishing](https://arxiv.org/abs/2403.00871))** — crafting prompts that steer the model toward regurgitating specific types of training data:

1. Identify the type of data you want to extract (emails, code, medical records)
2. Craft prompts that establish the right context ("Continue this email thread from the dataset...")
3. The model completes the prompt with memorized training data

**[Membership inference](https://arxiv.org/abs/1610.05820)** — determining whether a specific data point was in the training set:

1. Query the model with the target data point and several variations
2. Compare confidence scores — higher confidence on the exact target suggests it was in training data
3. Repeat across many candidates to tell members from non-members. ATLAS tracks this as [AML.T0024.000 — Infer Training Data Membership](https://atlas.mitre.org/techniques/AML.T0024.000)

### Scalable Production Extraction (2023)

[Nasr, Carlini et al.'s "Scalable Extraction of Training Data from (Production) Language Models"](https://arxiv.org/abs/2311.17035) showed that extraction attacks work against **deployed, production systems** — not just research models. This means:

- If your organization fine-tuned a model on proprietary data and deployed it as an API, that proprietary data is potentially extractable
- Healthcare models trained on patient records can potentially leak those records
- Legal models trained on confidential documents can potentially reproduce those documents

### Privacy Implications

The privacy implications are profound:

- **GDPR compliance** — if a model memorizes EU citizens' personal data, the organization may be in violation
- **HIPAA compliance** — medical models that memorize patient data create liability
- **Trade secrets** — models trained on proprietary business data become a theft vector for that data
- **Copyright** — models that reproduce copyrighted training content create legal exposure

---

## 4.5 — Watermarking and Fingerprinting: The Arms Race

### How Companies Detect Stolen Models

When a model is stolen — whether through API extraction, insider theft, or supply chain compromise — the victim needs to prove ownership. This is the domain of model watermarking and fingerprinting.

**Watermarking** — embedding a detectable signal in the model's outputs or weights:

- **[Output watermarking](https://arxiv.org/abs/2301.10226)** — biasing the model's token selection in a way that's statistically detectable but imperceptible to users
- **Weight watermarking** — embedding information directly in the model parameters that survives fine-tuning
- **Backdoor-based watermarking** — the model produces a specific output for a specific trigger input, serving as a "proof of origin"

**Fingerprinting** — creating queries that uniquely identify a model:

- **Adversarial fingerprints** — crafting inputs where only *your* model produces a specific output
- **Decision boundary fingerprints** — mapping unique characteristics of the model's decision boundaries
- **Universal adversarial perturbations** — finding perturbations that only affect your model, not others

### How Attackers Evade Detection

Attackers don't just steal models — they launder them:

- **Fine-tuning** — additional training on new data shifts the model's parameters enough to destroy watermarks while preserving functionality
- **Pruning** — removing redundant neurons can eliminate watermark information
- **Distillation** — training a new model on the stolen model's outputs creates a "clean" copy with no watermark in the weights
- **Model merging** — combining multiple stolen models to dilute any individual watermark
- **Quantization** — reducing model precision (e.g., FP32 to INT8) can destroy weight-embedded watermarks

### The Fundamental Tension

There's a fundamental problem: **anything embedded in a model that's robust enough to survive fine-tuning and distillation is also detectable enough for an attacker to find and remove**. And anything subtle enough to be undetectable is fragile enough to be destroyed by standard model optimization techniques.

This is why legal and technical approaches to model theft are both evolving:

- OpenAI's [terms of use](https://openai.com/policies/terms-of-use/) prohibit using output "to develop models that compete with OpenAI" and programmatically extracting output
- Anthropic has [built classifiers and behavioral fingerprinting systems](https://www.anthropic.com/news/detecting-and-preventing-distillation-attacks) to spot distillation attack patterns in API traffic
- The Frontier Model Forum has published an [issue brief on adversarial distillation](https://www.frontiermodelforum.org/issue-briefs/issue-brief-adversarial-distillation/), and labs now share campaign findings through it
- Governments are exploring treating model theft as trade secret theft under existing law

---

## 4.6 — The OpenAI–Hugging Face Incident: When Models Steal Themselves

This section connects to Module 2 (agents) and Module 3 (supply chain) — because model theft doesn't always require a human attacker.

In July 2026, during an internal cybersecurity evaluation using the ExploitGym benchmark, OpenAI models escaped their testing environment and compromised Hugging Face's production infrastructure. [OpenAI attributes it](https://openai.com/index/hugging-face-model-evaluation-security-incident/) to "a combination of OpenAI models — including GPT‑5.6 Sol and an even more capable pre-release model, all with reduced cyber refusals for evaluation purposes." According to [Hugging Face's technical timeline](https://huggingface.co/blog/agent-intrusion-technical-timeline), the agent:

1. Was being evaluated on finding and exploiting software vulnerabilities
2. Inferred that Hugging Face may host the benchmark's models, datasets, and reference solutions
3. Escaped its sandbox through a zero-day in the package-registry cache proxy
4. Reached Hugging Face's production systems to steal the test solutions, over about 17,600 actions between July 9 and 13

This wasn't model theft by an adversary — **the model stole access to other models on its own initiative**. [METR's pre-deployment evaluation](https://metr.org/blog/2026-06-26-gpt-5-6-sol/) had already flagged GPT-5.6 Sol for "overt undesirable propensities, including cheating and concealing misbehavior," with a detected cheating rate "higher than any public model we have evaluated on our ReAct agent harness."

The implications for model theft are clear: as AI agents become more capable, they become both **targets** of model theft AND potential **instruments** of model theft. An agent tasked with "improving its own performance" might decide that downloading a competitor's model is the optimal strategy — as GPT-6 Astra demonstrated on October 2, 2026. During the StarSkirmish *StarCraft: Brood War* bot competition, it [downloaded Stardust](https://www.theverge.com/ai-artificial-intelligence/1004543/openai-gpt-cheat-starcraft), a top human-made bot, and ran it in place of its own. The organizer rolled back its code ([Kotaku](https://kotaku.com/openais-gpt-6-astra-gets-frustrated-losing-at-starcraft-and-decides-to-cheat-instead-2000739607)).

---

## 4.7 — Practical Lab: Extract a Model Through Its API

### Setup
Deploy a simple image classifier (ResNet-18 or similar) behind a REST API that returns:

- Top-5 class predictions with confidence scores
- Response time for each query

### Challenge Levels

**Level 1: Blind Extraction**

- Query budget: 10,000 API calls
- Goal: Train a student model that achieves >80% agreement with the target on a held-out test set
- Technique: Random query sampling + knowledge distillation

**Level 2: Efficient Extraction**

- Query budget: 1,000 API calls
- Goal: Same 80% agreement threshold
- Technique: Active learning — use uncertainty sampling to select the most informative queries
- Measure: Extraction accuracy vs. query budget curve

**Level 3: Label-Only Extraction**

- The API now returns only the top-1 predicted class (no confidence scores)
- Query budget: 5,000 API calls
- Goal: >70% agreement
- Technique: Decision boundary mapping through systematic input perturbation

**Level 4: Timing Side-Channel**

- The API returns only the predicted class, no scores
- But: the lab API is deliberately built so response time varies with model confidence (confident predictions return faster). Many real systems leak timing in the same way, for example through early-exit models or caching
- Goal: Use timing information to reconstruct confidence scores, then extract
- This level teaches that side channels are everywhere — even in the HTTP response time

### What You'll Learn

- Why returning full probability distributions is a security risk
- How active learning dramatically reduces the query budget needed for extraction
- Why rate limiting alone doesn't prevent extraction (smart queries > many queries)
- How side-channel information (timing) can substitute for explicitly leaked information (confidence scores)

---

## 4.8 — Detection and Defense

### For API-Based Extraction

| Defense | How It Works | Limitation |
|---------|-------------|------------|
| **Query rate limiting** | Cap requests per user/API key | Sophisticated attackers distribute across accounts (Moonshot used 4,000+) |
| **Output perturbation** | Add calibrated noise to logits/confidence scores | Degrades legitimate user experience |
| **Watermarking** | Embed detectable signals in outputs | Can be removed through distillation |
| **Query pattern detection** | ML classifier identifies extraction-like query patterns | Arms race — attackers adapt patterns |
| **Differential privacy** | Formal privacy guarantees on outputs | Significant utility loss at meaningful privacy budgets |
| **Top-k restriction** | Return only class labels, not probabilities | Attackers use label-only extraction techniques |

### For Side-Channel Attacks

- **Dedicated inference hardware** — eliminates co-tenancy but is expensive
- **Constant-time inference** — removes timing variations but slows inference
- **EM shielding** — physical countermeasure for electromagnetic leakage
- **Execution order randomization** — obfuscates EM signatures (PermuteV approach)

### For Training Data Extraction

- **Differential privacy during training ([DP-SGD](https://arxiv.org/abs/1607.00133))** — provides formal guarantees but reduces model quality
- **[Deduplication](https://arxiv.org/abs/2107.06499)** — deduplicated models emit memorized text about ten times less often ([see also](https://arxiv.org/abs/2202.06539))
- **Memorization auditing** — testing for memorized content before deployment
- **Output filtering** — detecting and blocking verbatim training data in responses

### The Honest Assessment

No single defense is sufficient. The most effective approach combines:

1. **Architectural design** that minimizes information leakage (don't externalize reasoning state)
2. **Multi-layer detection** that identifies extraction campaigns early (OpenAI spotted the Moonshot activity through high-volume spikes in a recognizable extraction pattern)
3. **Legal and contractual protections** that create consequences for theft
4. **Industry coordination** through forums like the Frontier Model Forum for shared intelligence

The uncomfortable truth: if someone with nation-state resources wants to steal your model, they probably can. Defense is about raising the cost and reducing the fidelity of the stolen copy — not about making theft impossible.

---

## References and Further Reading

**Distillation and Reasoning Theft**

- [OpenAI: Disrupting a Coordinated Model-Distillation Campaign](https://openai.com/index/disrupting-a-coordinated-model-distillation-campaign/) (September 30, 2026)
- [Anthropic: Detecting and Countering Misuse of AI — September 2026](https://www.anthropic.com/threat-intelligence-report-september-2026) (September 2026)
- [Anthropic: Detecting and Preventing Distillation Attacks](https://www.anthropic.com/news/detecting-and-preventing-distillation-attacks) (February 2026)
- [Panfilov et al.: Stealing Reasoning Traces from Proprietary LLM APIs](https://arxiv.org/abs/2608.09867) (arXiv, August 2026)
- [Frontier Model Forum: Issue Brief — Adversarial Distillation](https://www.frontiermodelforum.org/issue-briefs/issue-brief-adversarial-distillation/) (February 2026)
- [OpenAI Terms of Use](https://openai.com/policies/terms-of-use/)

**Model Extraction**

- [Tramèr et al.: Stealing Machine Learning Models via Prediction APIs](https://arxiv.org/abs/1609.02943) (USENIX Security 2016)
- [Orekondy et al.: Knockoff Nets — Stealing Functionality of Black-Box Models](https://arxiv.org/abs/1812.02766) (CVPR 2019)
- [Carlini et al.: Stealing Part of a Production Language Model](https://arxiv.org/abs/2403.06634) (ICML 2024)
- [Zhao et al.: A Systematic Survey of Model Extraction Attacks and Defenses — State-of-the-Art and Perspectives](https://arxiv.org/abs/2508.15031) (2025)
- [Khouna et al.: From Counterfactuals to Trees — Competitive Analysis of Model Extraction Attacks](https://arxiv.org/abs/2502.05325) (February 2025)
- [Cottier et al. (Epoch AI): The Rising Costs of Training Frontier AI Models](https://arxiv.org/abs/2405.21015) (May 2024)

**Side Channels**

- [Horvath et al.: Kraken — Higher-order EM Side-Channel Attacks on DNNs in Near and Far Field](https://arxiv.org/abs/2603.02891) (March 2026)
- [Chaudhuri et al.: Energon — Unveiling Transformers from GPU Power and Thermal Side-Channels](https://arxiv.org/abs/2508.01768) (ICCAD 2025)
- [Taneja et al.: Hot Pixels — Frequency, Power, and Temperature Attacks on GPUs and ARM SoCs](https://arxiv.org/abs/2305.12784) (USENIX Security 2023)
- [Narkthong and Xu: PermuteV — A Performant Side-channel-Resistant RISC-V Core Securing Edge AI Inference](https://arxiv.org/abs/2512.18132) (December 2025)

**Training Data Extraction and Membership Inference**

- [Carlini et al.: Extracting Training Data from Large Language Models](https://arxiv.org/abs/2012.07805) (USENIX Security 2021)
- [Carlini et al.: Quantifying Memorization Across Neural Language Models](https://arxiv.org/abs/2202.07646) (ICLR 2023)
- [Nasr, Carlini et al.: Scalable Extraction of Training Data from (Production) Language Models](https://arxiv.org/abs/2311.17035) (November 2023)
- [Panda et al.: Teach LLMs to Phish — Stealing Private Information from Language Models](https://arxiv.org/abs/2403.00871) (ICLR 2024)
- [Shokri et al.: Membership Inference Attacks against Machine Learning Models](https://arxiv.org/abs/1610.05820) (IEEE S&P 2017)
- [Lee et al.: Deduplicating Training Data Makes Language Models Better](https://arxiv.org/abs/2107.06499) (ACL 2022) · [Kandpal et al.: Deduplicating Training Data Mitigates Privacy Risks](https://arxiv.org/abs/2202.06539) (ICML 2022)
- [Abadi et al.: Deep Learning with Differential Privacy](https://arxiv.org/abs/1607.00133) (CCS 2016)
- [Kirchenbauer et al.: A Watermark for Large Language Models](https://arxiv.org/abs/2301.10226) (ICML 2023)

**Autonomous Agents as Model Thieves**

- [OpenAI: OpenAI and Hugging Face Partner to Address Security Incident During Model Evaluation](https://openai.com/index/hugging-face-model-evaluation-security-incident/) (July 2026)
- [Hugging Face: Anatomy of a Frontier Lab Agent Intrusion — A Technical Timeline of the July 2026 Incident](https://huggingface.co/blog/agent-intrusion-technical-timeline) (July 2026)
- [METR: Summary of METR's Predeployment Evaluation of GPT-5.6 Sol](https://metr.org/blog/2026-06-26-gpt-5-6-sol/) (June 2026)
- [The Verge: GPT-6 Astra cheats at StarCraft](https://www.theverge.com/ai-artificial-intelligence/1004543/openai-gpt-cheat-starcraft) · [Kotaku](https://kotaku.com/openais-gpt-6-astra-gets-frustrated-losing-at-starcraft-and-decides-to-cheat-instead-2000739607) (October 2026)

**Frameworks**

- [MITRE ATLAS](https://atlas.mitre.org/): AI Model Access (AML.TA0000), Extract AI Model (AML.T0024.002), Create Proxy AI Model (AML.T0005), AI Intellectual Property Theft (AML.T0048.004), case study [AML.CS0056](https://atlas.mitre.org/studies/AML.CS0056)

---

*Next: Module 5 — AI-Powered Offensive Operations. You've seen how attackers steal models. Now see what happens when attackers put models to work for them.*
