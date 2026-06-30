# A2A (Agent-to-Agent Protocol)

**Aliases:** A2A, Google A2A, Linux Foundation A2A, agent interoperability protocol, cross-vendor agent protocol
**Category:** Protocols
**Sources:**
[Google — Announcing the Agent2Agent Protocol (Apr 9, 2025)](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) ·
[A2A official site](https://a2a-protocol.org/) ·
[A2A specification](https://a2a-protocol.org/latest/specification/) ·
[A2A GitHub (Linux Foundation)](https://github.com/a2aproject/A2A) ·
[Google A2A announcement on Vertex](https://cloud.google.com/blog/products/ai-machine-learning/introducing-agent2agent-protocol)

---

## Problem

> [!TIP]
> **ELI5.** [MCP](mcp.md) standardized agent-to-TOOL communication. But what about agent-to-AGENT? Real multi-agent systems involve many agents — built by different teams, on different frameworks (LangGraph, CrewAI, AutoGen, Anthropic's, Google's, custom), running on different infrastructure — that need to discover each other, negotiate capabilities, hand off tasks, and collaborate. Without a standard, every multi-agent integration was bespoke. **A2A** (Google, April 9, 2025) is the standard for this. Agents publish "agent cards" (capability descriptions); peers discover and call them; tasks track long-running work with status updates. Launched with 50+ partners (SAP, Salesforce, ServiceNow, Cohere, LangChain, MongoDB, etc.). Donated to the Linux Foundation in June 2025. **Complementary to MCP** — agents typically use MCP for tools AND A2A for sub-agents.

The April 9, 2025 [Google announcement](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) framed it: "Agent2Agent Protocol allows AI agents to communicate with each other, securely exchange information, and coordinate actions on various enterprise platforms or applications."

Why agent-to-agent needed its own protocol:

- **MCP is asymmetric** (client/server). A2A peers are symmetric — either can request, either can respond.
- **Agent-to-agent interactions are conversational and long-running** (tasks span minutes-to-days). MCP is more RPC-shaped.
- **Agents need discovery** ("what specialized agents are out there for my domain?"). MCP doesn't address this.
- **Multi-agent workflows cross trust boundaries** (different vendors, different orgs). Need standardized auth + identity.
- **Tasks have rich lifecycle** (planning, in-progress, blocked, requires-input, completed, failed). Different from MCP's request-response.

The launch had unusual industry backing — Google was joined by 50+ partners at launch including major enterprise software (SAP, Salesforce, ServiceNow, Atlassian, Workday) and AI vendors (Cohere, LangChain, MongoDB, Box). In June 2025 Google donated the protocol to the Linux Foundation, signaling neutral governance.

As of 2026, A2A is less mature than MCP (newer, less-deployed) but adoption is growing fast. Most production multi-agent systems by mid-2026 are starting to standardize on A2A for cross-agent communication.

## How it works

> [!TIP]
> **ELI5.** Each agent exposes itself via an "agent card" at `/.well-known/agent-card.json` — like a robots.txt for agents. The card describes capabilities, skills, auth requirements. Discovery means fetching that URL. Communication is task-oriented: open a task, exchange messages within it, receive structured artifacts (the work product), see status updates streamed back. Two agents can be from different vendors / frameworks / clouds — A2A is the lingua franca.

![A2A](../diagrams/svg/a2a.svg)

### The core concepts

**Agent Card.** A JSON document describing an agent:
- Name, description, version
- Capabilities (supported features: streaming, push notifications, file inputs, etc.)
- Skills (named, parameterized capabilities; like MCP prompts but agent-level)
- Authentication scheme (OAuth, API key, OIDC, etc.)
- Endpoint URLs

Served at `https://your-agent.example.com/.well-known/agent-card.json`. Discoverable like an OpenAPI spec.

**Task.** A unit of work, with lifecycle:
- `submitted` → `working` → `input-required` / `auth-required` → `completed` / `failed` / `canceled`
- Long-running by design (hours-to-days is normal)
- Identified by unique task ID; resumable
- May contain multiple messages and artifacts

**Message.** A conversational exchange within a task:
- Sender role (user / agent)
- Parts (text, structured data, file references, function calls)
- Streaming chunks for incremental delivery

**Artifact.** A structured output produced by an agent:
- Named, typed
- May be the final deliverable, or an intermediate work product
- Versioned

### A typical A2A interaction

1. **Discover**: client agent fetches the target agent's card.
2. **Authenticate**: client obtains auth (OAuth flow, API key, etc.).
3. **Open task**: client POSTs to the agent's endpoint with initial message.
4. **Stream**: agent processes; pushes status updates and partial artifacts via SSE.
5. **Input as needed**: agent may ask for clarification (`input-required` state); client responds.
6. **Complete**: agent returns final artifact(s); task ends.

The pattern is request-stream-respond, not strict request-response. This fits real agent work (which is rarely a one-shot call).

### How A2A differs from MCP

| Aspect | MCP | A2A |
|---|---|---|
| **Endpoints** | Client/server (asymmetric) | Peer/peer (symmetric) |
| **Use case** | Agent → tool | Agent → agent |
| **Interaction shape** | RPC (request → response) | Task (long-lived, conversational) |
| **Discovery** | Client-configured servers | `/.well-known/agent-card.json` |
| **Outputs** | Tool results | Tasks, messages, artifacts |
| **Lifecycle** | Per-call | Multi-step task lifecycle |
| **Streaming** | Optional (SSE) | First-class (SSE) |
| **Maturity** | Mature (2024+); ubiquitous | Newer (2025+); growing fast |

They're **complementary**, not competitive. A typical 2026 agent stack:
- Uses MCP for its tools (filesystem, GitHub, DB, etc.)
- Uses A2A to call specialist sub-agents (research agent, coding agent, scheduling agent)
- Same agent may be both an A2A client (calling sub-agents) and an A2A server (exposing itself for callers)

### Where A2A is used

Real production use cases (early adopters):

- **Cross-org workflows**: an enterprise agent in vendor A's platform calls a specialist agent in vendor B's platform.
- **Multi-cloud agents**: a Google Vertex agent collaborates with an AWS Bedrock agent and an Azure agent.
- **Specialist agent libraries**: enterprises build a "research agent," "coding agent," "compliance agent" each registered via A2A; other agents discover and call them.
- **Agent marketplaces**: vendors publish agents discoverable via A2A; customers consume via standard protocol.
- **Long-running tasks**: research, deep analysis, document generation that take hours.

### Authentication and trust

Cross-agent interactions cross trust boundaries. A2A includes:

- **OAuth 2.0 / OIDC** for token-based auth.
- **API keys** for simpler cases.
- **mTLS** for high-security deployments.
- **Capability scopes** — each token grants specific capabilities, not full agent access.

The discovery layer (`/.well-known/agent-card.json`) is itself unauthenticated by design — agents need to be discoverable without auth so peers can find them.

### Engineering details

- **Transport**: HTTP + Server-Sent Events for streaming; standard REST verbs.
- **Specification**: open under Linux Foundation; reference implementations in Python, Go, TypeScript.
- **Task IDs**: client-generated UUIDs typically; allow resumption across sessions.
- **Push notifications**: optional webhook URLs for async update delivery.
- **Schema**: JSON Schema for messages and artifacts.
- **Versioning**: protocol version negotiated on connection; agent card includes supported versions.

### Composition with MCP

The canonical 2026 stack:

```
User
  ↓
Top-level agent (LangGraph + Anthropic)
  ↓ uses MCP for:
    - filesystem server
    - GitHub server
    - internal DB server
  ↓ uses A2A for:
    - research sub-agent (running on Google Vertex)
    - compliance sub-agent (running internally)
    - scheduling sub-agent (vendor SaaS)
```

The top-level agent doesn't care that the research sub-agent is on a different cloud, different framework, different vendor. A2A makes it just-an-agent.

### Adoption and ecosystem (2025-2026)

**Launch (Apr 2025)**:
- 50+ partners on day one
- Enterprise software: SAP, Salesforce, ServiceNow, Atlassian, Workday, Adobe
- AI vendors: Cohere, LangChain, LlamaIndex, MongoDB, Box, JFrog
- Consulting: Accenture, Capgemini, Deloitte, PwC, Wipro

**Donation to Linux Foundation (Jun 2025)**:
- Spec governed by neutral foundation
- IBM, Microsoft, Cisco joined post-donation
- Symbolic of "this is industry infrastructure, not Google product"

**By 2026**:
- A2A SDKs across all major frameworks
- Google Vertex Agent Builder ships A2A natively
- LangChain / LangGraph have A2A integrations
- Anthropic supports A2A indirectly (via partners)
- Many enterprise agent vendors expose A2A endpoints

Adoption is slower than MCP (which had a one-year head start and a simpler use case) but accelerating.

### Anti-patterns

- **A2A for tools**. Use MCP. A2A's task lifecycle is overkill for "read this file."
- **MCP for sub-agents**. MCP doesn't have task lifecycle; long-running sub-agent work is awkward.
- **Unauthenticated A2A endpoints** exposing sensitive operations. Auth is non-optional in production.
- **Skipping the agent card**. Without it, no discovery; A2A's main benefit gone.
- **Building a custom protocol instead.** Reinventing what A2A standardizes is wasted effort.
- **A2A endpoints without rate limiting / quotas.** Easy to DoS.
- **Cross-agent calls without observability.** Hard to debug multi-agent failures.
- **Trusting external agent outputs blindly.** [Lethal trifecta](../sec/lethal-trifecta.md) applies — external agent output is untrusted input.

### When NOT to use A2A

- **Single-agent apps.** No need for inter-agent protocol.
- **All sub-agents are first-party / same-process.** Use a function call instead.
- **Latency-critical paths.** A2A has HTTP + SSE overhead.
- **Closed-system multi-agent.** Internal frameworks may be simpler.
- **Maturity-sensitive deployments.** A2A is newer than MCP; expect some rough edges.

### Trajectory through 2026 and beyond

- **Convergence of MCP + A2A**: the two protocols increasingly co-exist; many agents speak both.
- **Agent marketplaces**: A2A discovery enables agent app stores.
- **Cross-org compliance**: standardized agent-to-agent auth simplifies B2B AI workflows.
- **A2A-native frameworks**: emerging — Vertex Agent Builder, others.
- **Possible MCP integration**: rumors of formal MCP/A2A bridges; may converge into broader "agent stack" specs.

For app developers building multi-agent systems in 2026, A2A is increasingly the default. For single-agent apps, MCP suffices.

## Variants & related patterns

- [**MCP (Model Context Protocol)**](mcp.md) — the agent-to-TOOL standard; complementary.
- [**Multi-agent orchestration**](../agt/multi-agent-orchestration.md) — A2A is the protocol layer.
- [**Sub-agent architectures**](../agt/sub-agent-architectures.md) — A2A enables vendor-agnostic sub-agents.
- [**Orchestrator-workers**](../wf/orchestrator-workers.md) — workers may be A2A-discoverable agents.
- [**Self-directed swarms**](../agt/self-directed-swarms.md) — A2A could underpin true cross-vendor swarms.
- [**Maker-checker**](../agt/maker-checker.md) — checker may be a separate A2A agent.
- [**Containment & blast radius**](../sec/containment-blast-radius.md) — A2A peer is untrusted code.
- [**Lethal trifecta**](../sec/lethal-trifecta.md) — same security model.
- **Service mesh** (system design) — analogous infrastructure pattern at the service-to-service layer.
- **OpenAPI / gRPC** — comparable but for general HTTP / RPC, not agent-specific.

## When NOT to use

- **Single-agent apps.**
- **All sub-agents are in-process function calls.**
- **Highly latency-sensitive sub-agent calls.**
- **Closed system with bespoke protocol working well.**

## Implementations

| SDK / framework | A2A support |
|---|---|
| **A2A reference Python SDK** | Official (Linux Foundation) |
| **A2A reference TypeScript SDK** | Official (Linux Foundation) |
| **A2A Go / Rust / Java** | Community / partners |
| **Google Vertex Agent Builder** | Native (out of box) |
| **LangChain / LangGraph** | A2A integration |
| **LlamaIndex** | A2A integration |
| **CrewAI, AutoGen** | A2A adapters available |
| **Cohere Compass + Agents** | A2A-compatible |
| **Anthropic Claude (via integrations)** | A2A via partners |
| **OpenAI Agents SDK** | Custom A2A bridges (not native at time of writing) |

## Companies / products using A2A

- **Google** ✅ — created and ships A2A on Vertex.
- **SAP, Salesforce, ServiceNow, Atlassian, Workday** ✅ — launch partners; integrations rolling out.
- **Adobe, Box, MongoDB, JFrog** ✅ — launch partners.
- **Cohere, LangChain, LlamaIndex** ✅ — AI vendor adoption.
- **IBM, Microsoft, Cisco** ✅ — joined post-Linux Foundation donation.
- **Accenture, Capgemini, Deloitte, PwC, Wipro** ✅ — consulting partners implementing for clients.
- **Most enterprise multi-agent deployments (2025-2026)** ⚠ — adopting A2A as cross-vendor lingua franca.

## Further reading

- [Announcing the Agent2Agent Protocol](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) — Google Apr 9, 2025 (launch)
- [A2A official site](https://a2a-protocol.org/) — current spec, docs
- [A2A specification](https://a2a-protocol.org/latest/specification/) — formal spec
- [A2A GitHub repo](https://github.com/a2aproject/A2A) — Linux Foundation home
- [Google Cloud A2A blog](https://cloud.google.com/blog/products/ai-machine-learning/introducing-agent2agent-protocol)
- [The new agent stack: MCP, A2A, and beyond](https://huyenchip.com/2025/06/agents.html) — Chip Huyen
- [LangChain A2A integration](https://python.langchain.com/) — once available
- [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol) — for MCP comparison context

---

*Diagram source: [`../diagrams/src/a2a.d2`](../diagrams/src/a2a.d2)*
