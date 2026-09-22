
---

## 1. What it is and why it exists

Foundry is Microsoft's enterprise platform for building, grounding, and governing AI apps and agents. Before it, you'd stitch together Azure OpenAI, Azure AI Services, Azure ML, AI Search, and separate safety and monitoring tools. Foundry consolidates several previous Azure AI services and tools into a unified platform.

You can think of it as three layers:

1. **Models**: what generates or reasons.
2. **Agents and apps**: orchestration, tools, knowledge, memory.
3. **Trust layer**: evaluation, observability, safety, security, governance.

## 2. Resource and project architecture

This is the part people most often get wrong, because it's Azure resource plumbing.

- **Foundry resource**: a top-level resource for governance, with projects for development isolation and connected Azure services for storage, search, and secrets. It's where you set networking, RBAC, policy, and model deployments.
- **Project**: a workspace inside the resource. Projects let teams prototype in a preconfigured environment, reusing existing model deployments and connections without repeated IT setup. They hold files, agents, evaluations, and other artifacts.
- **Connections / connected resources**: Azure services such as Storage, Key Vault, and Azure AI Search that the Foundry resource references.
- **Hub-based vs. Foundry projects (legacy vs. current)**: the older model was Hub → Projects, with the hub holding shared security settings. Hub-based projects are still accessible in the Foundry (classic) portal, but new investment is going into Foundry projects in the new portal.
- **Migrating from Azure OpenAI**: you can upgrade an Azure OpenAI resource to a Foundry resource while keeping its endpoint, API keys, and state. Existing custom Azure policies and RBAC actions continue to apply.

Because it's a normal Azure resource, you get standard **Azure RBAC**, **managed identities**, **private endpoints / VNet integration**, **Azure Policy**, and **Key Vault**.

## 3. The model layer

**Catalog.** Foundry offers over 11,000 foundational, open, reasoning, multimodal, and industry-specific models, spanning OpenAI, Anthropic, Meta, Google, xAI, Hugging Face, and Microsoft's own MAI family. Model counts and providers change quickly, so treat the numbers as approximate.

**Deployment.** You don't call a model until you _deploy_ it, which creates an endpoint with a name, version, quota, and its own content-filter settings. The main types:

- **Standard (pay-per-token)**: shared capacity, with regional or "Global" routing for higher availability and quota. It's the default for most work.
- **Provisioned throughput (PTU)**: reserved capacity for predictable latency and throughput, priced per unit of capacity.
- **Batch**: queues inference requests for asynchronous execution at reduced per-token pricing.
- **Managed compute**: for some open models, you host on dedicated VMs and pay for the compute, not the tokens.

The "models as a service" idea is that Microsoft hosts the model and you consume it through an API. This is considered a MaaS deployment.

**Customization**, from cheapest to most involved:

1. Prompt engineering.
2. RAG (retrieval-augmented generation).
3. Fine-tuning (supervised, and for some models preference or reinforcement-style tuning).
4. Distillation (a big model teaches a smaller one).

Use the cheapest option that solves the problem. RAG changes what the model _knows_, and fine-tuning changes how it _behaves_.

**Model router / selection.** Foundry offers benchmarking and comparison tools, plus routing that picks a model per request based on cost and quality trade-offs.

## 4. Building: knowledge, agents, tools

### Grounding / RAG

The pattern is: chunk and index your data, retrieve relevant chunks at query time, and pass them to the model with the prompt. **Azure AI Search** is the usual retrieval engine. It supports full-text, semantic, vector, and hybrid search, and serves as the knowledge store for the RAG pattern. Hybrid search (keyword plus vector, then semantic re-ranking) is generally the strongest default. Foundry IQ provides knowledge bases that ground agents in sources like OneLake and SharePoint over MCP.

### Foundry Agent Service

An **agent** is a model plus instructions plus tools plus (optionally) knowledge and memory, running in a loop: reason, call a tool, observe the result, repeat.

- In a basic chat architecture, Agent Service orchestrates fetching grounding data from AI Search and other tools, then passes it with the prompt to the deployed model.
- **Prompt agents** are defined declaratively. The system prompt, temperature, top_p, and constrained knowledge connections define how the agent behaves for all requests.
- Beyond prompt agents, there are **workflow** agents (multi-step, multi-agent orchestration) and **hosted** agents (your own code, run in managed containers). Agent workloads run inside the platform's container infrastructure, with virtual network integration for isolated scenarios.
- **Tools**: built-in ones (code interpreter, file/web search, and similar), OpenAPI tools, and **MCP (Model Context Protocol)** servers. One source counts 1,400+ MCP-enabled, built-in, and OpenAPI tools agents can call.
- **Memory** is a preview capability, so check current status before relying on it.

### Code-level frameworks

You interact through the Foundry SDKs and REST APIs, and often through orchestration frameworks such as the Microsoft Agent Framework (the successor line to Semantic Kernel and AutoGen), LangChain, or others. Framework choice matters less than the concepts above.

## 5. Evaluation and observability

This is what separates a demo from a product.

- **Evaluation** means scoring outputs against criteria. Evaluations invoke model endpoints, compare outputs against grading criteria, and store results within the project scope. Common metric families:
    - _Quality_: groundedness, relevance, coherence, fluency, similarity.
    - _Safety_: hate, violence, self-harm, sexual content, jailbreak susceptibility.
    - _Agent-specific_: did it pick the right tool, use the right arguments, complete the task?
    - Evaluators can be AI-assisted (an LLM as judge), code-based, or human.
- **Red teaming** is adversarial testing, automated or manual, to find safety gaps before users do.
- **Observability** means tracing every step of an agent run (prompts, tool calls, latency, tokens), typically via OpenTelemetry. Foundry lets you trace and evaluate agents and models and monitor them with built-in dashboards (preview).

The loop to remember is **evaluate offline before release, then monitor and evaluate continuously in production.**

## 6. Safety, security, and governance

**Content safety / guardrails.** Guardrails define the risks to detect, where to scan, and what to do when a risk is found. Intervention points include user input, output, tool calls (preview), and tool responses (preview). Content filters run inline with model requests and can be configured per deployment. Related features you should know by name:

- **Prompt Shields**: detect direct jailbreaks and indirect prompt injection (malicious instructions hidden in documents or tool output).
- **Groundedness detection**: flags claims unsupported by the source material.
- **Protected material detection**: flags known copyrighted text or code.

**Security**: Entra ID auth (prefer it over API keys), managed identities, RBAC roles separating those who _manage_ from those who _build_, private networking, customer-managed keys, and data-residency and no-training-on-your-data commitments.

**Governance**: Azure Policy, Microsoft Purview (data classification, DLP), Defender for Cloud (threat protection for AI workloads), and fleet-wide visibility over agents and models. Microsoft's pitch is consistent security, compliance, and policy controls across every agent.

## 7. Operations

- **Quotas and rate limits** are per deployment and region (tokens per minute, requests per minute). Design for 429 responses with retry and backoff.
- **Cost levers**: model choice, batch for non-urgent work, prompt and context length, caching, and PTU when traffic is steady and high.
- **Resilience**: multi-region deployments, gateway patterns (API Management in front of models for load balancing, auth, and chargeback), and fallbacks between models.
- **Environments and CI/CD**: separate dev, test, and prod projects. Treat prompts, agent definitions, and evaluation sets as versioned code.

## 8. Where Foundry sits among neighbouring services

|Service|Relationship|
|---|---|
|**Azure OpenAI**|Now a subset of Foundry's model offering.|
|**Azure AI Services** (Speech, Vision, Language, Document Intelligence, Content Understanding)|Prebuilt task-specific APIs, reachable from Foundry and usable as agent tools.|
|**Azure AI Search**|The retrieval backbone for RAG.|
|**Azure Machine Learning**|Classic ML: training, pipelines, MLOps. Foundry is oriented toward generative AI and agents.|
|**Copilot Studio**|Low-code agent building for business users. Foundry is the pro-code counterpart.|

## 9. The end-to-end mental model

1. **Set up**: a Foundry resource, a project, connections, and RBAC.
2. **Choose and deploy a model**: compare from the catalog, pick a deployment type.
3. **Ground it**: index data (AI Search, Foundry IQ).
4. **Build the agent or app**: instructions, tools, MCP, memory, and orchestration if multiple agents.
5. **Evaluate and red-team** offline.
6. **Apply guardrails and security**: filters, Prompt Shields, identity, networking.
7. **Deploy and scale**: quotas, PTU or batch, gateway, environments.
8. **Observe and iterate**: traces, continuous evaluation, cost tracking.

## Self-check questions

- What's the difference between a Foundry resource and a project?
- When would you choose RAG over fine-tuning?
- What are the four guardrail intervention points?
- Why is indirect prompt injection especially dangerous for agents with tools?
- Standard vs. provisioned vs. batch: which fits a nightly report job? A latency-critical chatbot?

**Caveat:** this platform changes fast. Several capabilities above (memory, some guardrail intervention points, dashboards) are marked preview in the docs, so check current status on Microsoft Learn before designing around them.

I can go deeper on any section, quiz you on it, or put this into a document you can keep. Which would help most?