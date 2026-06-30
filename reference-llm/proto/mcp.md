# Model Context Protocol (MCP)

**Aliases:** MCP, the USB-C of AI, Anthropic tool protocol, open agent-tool standard
**Category:** Protocols
**Sources:**
[Anthropic — Introducing the Model Context Protocol (Nov 25, 2024)](https://www.anthropic.com/news/model-context-protocol) ·
[Model Context Protocol official site](https://modelcontextprotocol.io/) ·
[MCP specification](https://spec.modelcontextprotocol.io/) ·
[Anthropic MCP servers repository](https://github.com/modelcontextprotocol/servers) ·
[OpenAI MCP adoption (Mar 26, 2025)](https://openai.com/) ·
production usage: Claude Desktop, Cursor, Continue, Zed, VS Code, OpenAI Agents SDK, Google Vertex, Microsoft Copilot Studio

---

## Problem

> [!TIP]
> **ELI5.** Every LLM application that wanted to call tools — read files, query a database, post to Slack, search GitHub, browse the web — had to *build its own integration layer*. Cursor, Claude Desktop, custom enterprise agents all re-implemented "talk to the filesystem," "talk to GitHub," "talk to Slack." Worse: a tool integration built for one app couldn't easily move to another. **MCP** (Anthropic, November 25, 2024) is the open standard that fixes this. Think "USB-C for AI tools": one protocol, many clients (LLM apps), many servers (tool providers). Write a tool server once; any MCP-aware client can use it. By late 2025, OpenAI, Google, and Microsoft had all adopted MCP — making it the de-facto agent-tool integration standard for the entire industry.

The coordination problem was acute by 2024. Every agent framework, every LLM app, every enterprise integration had a one-off tool layer. As tool ecosystems grew (a typical agent needed 10+ tools), the per-integration cost compounded.

Anthropic's [November 25, 2024 launch](https://www.anthropic.com/news/model-context-protocol) framed it directly: "AI assistants need to be able to interact with tools and data sources... MCP solves this by providing a universal, open standard for connecting AI systems to data sources."

The adoption was unusually fast. OpenAI announced MCP support in March 2025 (its [Agents SDK](https://platform.openai.com/docs/agents) included MCP from launch). Google added MCP to Vertex / Gemini API in April. Microsoft committed to MCP in Copilot Studio in May. By mid-2025 MCP was effectively universal — the rare case of an industry standard emerging from a single vendor and being adopted by all major competitors within months.

## How it works

> [!TIP]
> **ELI5.** Two roles. **Clients** are LLM apps (Claude Desktop, Cursor, your custom agent). **Servers** expose tools (filesystem, GitHub, your internal database). They talk over JSON-RPC — stdio for local, HTTP / SSE for remote. A server exposes three things: **resources** (read-only data the LLM can access), **tools** (functions the LLM can call), **prompts** (templates the client can use). A client can connect to many servers; a server can serve many clients. Anyone can publish a server; anyone can use one. Like npm for AI tool integrations.

![MCP](../diagrams/svg/mcp.svg)

### The three primitives

**1. Resources** — read-only data the client/LLM can access.
- File contents, database rows, API responses, web pages, etc.
- Listable, readable, optionally subscribable for updates.
- Identified by URIs (`file:///path`, `postgres://...`, `github://repo/issues`).
- Used like: "Read me the contents of this resource."

**2. Tools** — callable functions with typed inputs and outputs.
- The LLM picks a tool, supplies arguments, the server executes.
- Standard JSON Schema for parameter validation.
- Returns text, structured data, or rich content (images, etc.).
- Used like: "Call `create_github_issue` with `{title: "...", body: "..."}`"

**3. Prompts** — named, parameterized prompt templates the server exposes.
- Server can offer best-practice prompts for its capabilities ("here's how to ask the SQL agent").
- Client presents them in UI; user picks one; parameters filled in.
- Used like: a slash command in a chat UI.

These three cover most real tool-integration needs. The protocol also has sampling (server → client requests for completions) and roots (filesystem boundaries) as extensions.

### Transport options

MCP runs over multiple transports:

- **stdio**: server runs as a local subprocess of the client; communication via stdin/stdout. Most common for local tools (filesystem, git, local DBs).
- **HTTP + SSE (Server-Sent Events)**: server runs as an HTTP service; client connects over HTTP, receives async updates via SSE. Used for remote/cloud servers.
- **WebSocket**: real-time bidirectional. Less common but supported.

The protocol layer is JSON-RPC 2.0; the transport layer is pluggable.

### A typical interaction

1. **Client startup**: connects to one or more MCP servers (configured by user / app).
2. **Capability negotiation**: client and server announce supported features.
3. **List resources / tools / prompts**: client queries what's available.
4. **LLM call**: client constructs a prompt including available tools.
5. **LLM emits tool call**: client receives a tool-call decision from the model.
6. **Tool execution**: client sends the call to the server; server executes; returns result.
7. **LLM receives result**: client feeds it back to the model for the next step.
8. **Optional notifications**: server pushes updates ("this resource changed").

This is the same shape as traditional tool-using agents — what MCP standardizes is the *protocol between client and server*, not the agent loop itself.

### The MCP server ecosystem (2025-2026)

By 2026 the ecosystem includes hundreds of public MCP servers:

**Official servers (Anthropic):**
- Filesystem, GitHub, Git, Slack, Google Drive, Postgres, SQLite, Brave Search, Puppeteer, Memory, Fetch (HTTP), Sequential Thinking, AWS, Cloudflare, etc.

**Vendor-shipped servers:**
- Notion, Linear, Atlassian, Stripe, Zapier, Asana, Salesforce — many SaaS vendors ship official MCP servers exposing their APIs.

**Community servers:**
- Hundreds on github.com/modelcontextprotocol/servers and community lists.
- Database wrappers (Snowflake, BigQuery, MongoDB, Redis).
- Productivity (calendar, email, todoist).
- Dev tools (Docker, Kubernetes, npm).
- Search engines (Google, Bing, DuckDuckGo, Tavily).
- AI services (image gen, OCR, speech).

The "many servers, easy install" experience is similar to a package manager.

### Why this worked when other attempts didn't

Several factors:

- **Pre-existing tool-using pattern**. LLM apps were already calling tools; MCP only standardized the protocol, not the use case.
- **Single strong reference implementation** (Anthropic). One vendor shipped servers, clients, SDKs simultaneously.
- **Right level of abstraction**. Lower than "agents" (which are too app-specific) but higher than HTTP (which has no LLM semantics).
- **Open spec, multiple language SDKs**. Python, TypeScript, others from day one.
- **No lock-in by design**. Anthropic explicitly invited competitors.
- **Real value visible immediately**. Plug Claude Desktop into the GitHub MCP server; suddenly Claude can manage your repos.

The combination led to faster industry adoption than typical for a new standard.

### How MCP composes with other patterns

- **MCP + [code mode](../skills/code-mode.md)**: code-mode agents call MCP tools from inside sandboxed code. The big-leverage pattern of late 2025.
- **MCP + [skills](../skills/skill-md-format.md)**: SKILL.md may declare which MCP servers it needs.
- **MCP + [augmented LLM](../fnd/augmented-llm.md)**: tools (one of the three augmentations) flow through MCP.
- **MCP + [agent loop](../agt/agent-loop.md)**: tools in the loop come from MCP servers.
- **MCP + [single agent with tools](../agt/single-agent-with-tools.md)**: most modern instances use MCP for the tools.
- **MCP + [coding agents](../agt/coding-agents.md)**: file system / git / GitHub MCP servers are standard.

By 2026 it's hard to find a serious LLM app that doesn't speak MCP.

### Security considerations

MCP servers can read files, execute commands, talk to APIs — they're highly capable. Security model:

- **User opt-in per server.** Servers don't auto-install; user explicitly enables.
- **Capability scoping.** Servers should expose minimum needed (e.g., read-only filesystem MCP for browse, write MCP for explicit edit).
- **Auth flow** for OAuth-style servers (GitHub, Google).
- **Sandbox the server.** A malicious or compromised server gets the permissions you grant it.
- **The [lethal trifecta](../sec/lethal-trifecta.md) applies.** MCP servers exposing private data + interacting with untrusted content + with egress are dangerous.

This is the modern version of [containment & blast radius](../sec/containment-blast-radius.md) — apply the same discipline.

### Engineering details

- **Server config**: typically a JSON file (`claude_desktop_config.json`, `mcp.json`) listing servers + transport details.
- **Authentication**: per-server (OAuth tokens, API keys); MCP doesn't dictate the auth.
- **Tool naming conflicts**: when multiple servers offer the same tool name; client must disambiguate.
- **Latency**: stdio servers are fast; HTTP / SSE servers add network round-trip.
- **Error handling**: standard JSON-RPC error codes + MCP-specific extensions.
- **Streaming**: long-running tools stream results back (helpful for long jobs).
- **Versioning**: protocol is versioned; clients and servers negotiate on connection.

### Anti-patterns

- **Auto-enabling all servers.** Security risk; user surprise. Opt-in per server.
- **Skipping auth.** Many servers need real auth (GitHub token, etc.).
- **Untrusted MCP servers** from random GitHub repos. Vet like any code dependency.
- **MCP server with broad permissions.** Filesystem-everywhere is too much; scope to specific dirs.
- **Skipping the sandbox.** MCP server compromise = host compromise.
- **One server, many use cases.** Mega-servers are hard to audit.
- **No telemetry on tool calls.** You can't audit what the agent did.

### When NOT to use MCP

- **Single-tool apps** where the integration is so simple you don't need a protocol.
- **Extremely high-throughput** scenarios where JSON-RPC overhead matters.
- **Air-gapped environments** without MCP server availability.
- **Proprietary deep integrations** where MCP's generality is friction.
- **Internal RPC** for tool-using libraries you control end-to-end.

Even so, MCP often wins by default — being the standard is itself valuable.

## Variants & related patterns

- [**A2A (Agent-to-Agent Protocol)**](a2a.md) — MCP is agent→tool; A2A is agent→agent. Complementary.
- [**Code Mode**](../skills/code-mode.md) — code-mode agents use MCP servers as importable functions.
- [**SKILL.md format**](../skills/skill-md-format.md) — skills often declare MCP server dependencies.
- [**Augmented LLM**](../fnd/augmented-llm.md) — tools (one augmentation) flow through MCP.
- [**Agent loop**](../agt/agent-loop.md) — MCP supplies the tools.
- [**Single agent with tools**](../agt/single-agent-with-tools.md) — modern instances use MCP.
- [**Coding agents**](../agt/coding-agents.md) — heavy MCP server users.
- [**Containment & blast radius**](../sec/containment-blast-radius.md) — security model for MCP servers.
- [**Lethal trifecta**](../sec/lethal-trifecta.md) — MCP security concern.
- **OpenAPI** — comparable HTTP-API standard (orthogonal; MCP is for agent-tool, OpenAPI for HTTP service-service).
- **LSP (Language Server Protocol)** — Microsoft's earlier "USB-C for editor-compiler" — direct inspiration.

## When NOT to use

- **Tiny single-tool apps.**
- **Extreme high-throughput RPC** with JSON-RPC overhead.
- **Tightly-coupled internal tools** where bare function calls suffice.
- **Highly proprietary deep integrations** where MCP's generality is friction.

## Implementations

| SDK / language | Maintained by |
|---|---|
| **TypeScript** | Anthropic (official) |
| **Python** | Anthropic (official) |
| **Go** | Community |
| **Rust** | Community |
| **Java / Kotlin** | Community |
| **C# / .NET** | Community |
| **Swift** | Community |
| **Ruby** | Community |

| Client (MCP consumer) | Supports MCP |
|---|---|
| **Claude Desktop** | ✅ Native (Anthropic) |
| **Cursor** | ✅ Native |
| **Continue** | ✅ Native |
| **Zed** | ✅ Native |
| **VS Code (via extensions)** | ✅ Native + Copilot integration |
| **OpenAI Agents SDK** | ✅ Native (Mar 2025) |
| **OpenAI ChatGPT (Plus / Team)** | ⚠ Adoption rolling out |
| **Google Gemini / Vertex** | ✅ Native (Apr 2025) |
| **Microsoft Copilot Studio** | ✅ Native (May 2025) |
| **LangChain, LangGraph** | ✅ MCP integration |
| **LlamaIndex** | ✅ MCP integration |
| **Mastra, Vercel AI SDK** | ✅ Native MCP support |

## Companies / products using MCP

- **Anthropic** ✅ — created and ships MCP across Claude products.
- **OpenAI** ✅ — Agents SDK ships with MCP (Mar 2025).
- **Google** ✅ — Vertex AI, Gemini API support MCP.
- **Microsoft** ✅ — Copilot Studio supports MCP.
- **Cursor, Continue, Zed, VS Code (Copilot)** ✅ — MCP-native editors.
- **Cognition Devin** ⚠ — uses MCP servers for tools.
- **Replit Agent** ⚠ — MCP integration.
- **Block (Square)** ✅ — early MCP launch partner.
- **Notion, Linear, Atlassian, Stripe, Salesforce, SAP, ServiceNow** ✅ — ship official MCP servers.
- **AWS, Cloudflare** ✅ — official MCP servers for cloud resources.
- **GitHub, Snowflake, MongoDB, Postgres ecosystem** ✅ — official or community MCP servers.

## Further reading

- [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol) — Anthropic Nov 25, 2024 (canonical launch post)
- [Model Context Protocol official site](https://modelcontextprotocol.io/) — docs, spec, examples
- [MCP specification](https://spec.modelcontextprotocol.io/) — formal spec
- [Anthropic MCP servers repo](https://github.com/modelcontextprotocol/servers) — open-source servers
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — Anthropic Nov 2025
- [The new agent stack: MCP, A2A, and beyond](https://huyenchip.com/2025/06/agents.html) — Chip Huyen
- [OpenAI Agents SDK + MCP](https://platform.openai.com/docs/agents) — OpenAI docs

---

*Diagram source: [`../diagrams/src/mcp.d2`](../diagrams/src/mcp.d2)*
