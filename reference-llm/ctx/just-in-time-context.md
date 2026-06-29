# Just-in-Time Context

**Aliases:** JIT context, on-demand retrieval, agentic retrieval, lazy loading, autonomous exploration
**Category:** Context Engineering
**Sources:**
[Anthropic — Effective context engineering for AI agents (Sept 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
Claude Code's design (referenced in the same post) ·
Mirrors classic computer-architecture demand-paging

---

## Problem

> [!TIP]
> **ELI5.** Imagine packing for a trip and stuffing the suitcase with *every book you might possibly want to read*, just in case. You can barely close it, you can barely lift it, and at the destination you only read three. The alternative: pack a Kindle with a list of books and download them when you decide you want them. You travel light and only "pay" for what you actually use. JIT context is the Kindle approach for LLM agents.

The 2023-2024 default for giving an LLM access to private data was **pre-loaded retrieval**: embed the user query, fetch the top-K most-similar chunks from a vector store, stuff them all into the prompt, ask the model to answer. Modern RAG. The agent gets one shot — whatever the retrieval step grabbed up front is what it has to work with.

This works when:
- The query is short, single-shot, and well-scoped.
- The "relevant" data is small enough to fit comfortably in context.
- You don't need to follow leads or look up secondary references.

It breaks down when:
- The agent needs to **explore** — look at a file, decide based on what it found whether to look at *another* file, repeat.
- The relevant data is large but only a small slice matters per turn.
- The agent makes many tool calls and pre-loaded context becomes irrelevant by turn 10.
- Pre-loading bloats the context and triggers [context rot](context-rot.md).

The pattern that emerged in production 2025 agents: **don't pre-load. Carry addresses.** Keep lightweight identifiers — file paths, URLs, query templates, MCP server names — in context, and let the agent fetch the actual data through tools *only when it decides it needs them*. Anthropic calls this **just-in-time context**.

## How it works

> [!TIP]
> **ELI5.** Instead of stuffing all the docs into the prompt, you give the agent a *table of contents* (file paths, URLs, query templates) plus *tools to fetch* (read_file, search, query_db). The agent looks at the table of contents, decides what's relevant, fetches just that piece, uses it, and moves on. Like a human researcher with a library catalogue and the ability to walk to the right shelf — not someone who carries every book to the desk.

![Just-in-time vs pre-loaded](../diagrams/svg/jit-vs-preload.svg)

The mechanics are simple. The agent's context contains:

- **System prompt** (small, always-relevant)
- **Lightweight identifiers** for data that *might* be relevant: file paths, document IDs, URLs, MCP server names, SQL query templates
- **Tools** for retrieving the actual data: `read_file`, `grep`, `glob`, `search`, `query_db`, `mcp_call`

What the context does *not* contain: the actual contents of the data.

When the agent needs information, it calls a retrieval tool with a specific argument it derived from the identifiers (e.g., `read_file("src/auth/login.py")` after seeing `src/auth/` in a file listing). The tool returns *just the slice* the agent asked for. The agent uses it, possibly stores a note about it (see [structured note-taking](structured-note-taking.md)), and continues.

Compare to pre-loaded retrieval: instead of dumping `src/auth/login.py`, `src/auth/middleware.py`, `src/auth/utils.py`, `src/auth/tests/test_login.py`, ... into the prompt and hoping the model picks the right one, you put `src/auth/` in the prompt as a directory listing and let the agent decide.

### Why it works better than pre-loading

**Token efficiency.** A directory listing is tens of tokens. The files themselves can be thousands or millions. Loading only what the agent uses means an order-of-magnitude reduction in token spend for most agentic tasks. Anthropic reported a case (in their [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) post) where moving from pre-loaded tool definitions to JIT discovery reduced token usage **from 150K to 2K — a 98.7% reduction**.

**Mirrors human cognition.** Humans don't memorize entire codebases or document corpora; we use external organization (filesystems, inboxes, bookmarks, search) to retrieve on demand. LLM agents — especially when given filesystem tools — are excellent at this kind of demand-paging.

**Metadata becomes signal.** The presence of `test_utils.py` in a `tests/` folder implies a different purpose than the same file in `src/core_logic/`. Folder hierarchies, naming conventions, file sizes, timestamps — all of these are *cheap* context that help the agent reason about what to fetch without paying the token cost of fetching it. Anthropic explicitly calls these out: *"Folder hierarchies, naming conventions, and timestamps all provide important signals that help both humans and agents understand how and when to utilize information."*

**Enables progressive disclosure.** Once you're in JIT mode, you can layer the retrieval: first fetch a summary/index, then drill into a specific file, then read a specific function. See [`progressive-disclosure.md`](progressive-disclosure.md).

**Sidesteps stale indexing.** Pre-loaded retrieval depends on an index built at some point in the past. JIT fetches *live* — the filesystem (or database, or API) at query time. Critical for agents working in fast-moving environments.

### Claude Code as the reference implementation

Anthropic uses Claude Code as the canonical example. Its primitives:

- `glob` — find files matching a pattern (returns paths, not content)
- `grep` — find lines matching a pattern across files (returns matched lines + locations)
- `read_file` — fetch a specific file's contents (with optional line range)
- `Bash` — execute commands like `head`, `tail`, `wc`, `find` for ad-hoc data exploration

These let Claude Code work on **massive codebases without ever loading the full repo into context**. It explores: `glob("**/*.py")` → see the structure → `grep("def authenticate", "src/")` → narrow down → `read_file("src/auth/login.py", lines="50-100")` → focused content. Each step's output informs the next decision.

### Hybrid strategies

Pure JIT has a cost: runtime exploration is slower than pre-loaded data, because the agent makes more LLM calls (one per fetch decision). For well-known, always-relevant context, pre-loading wins. The 2026 best practice is **hybrid**:

![Hybrid: pre-load anchors, JIT the rest](../diagrams/svg/hybrid-context.svg)

Pre-load the small, always-relevant stuff: the system prompt, project conventions (`CLAUDE.md`-style files), the current task definition, a state summary. Carry **references** (file paths, MCP server names, query templates) for everything else. Let the agent fetch via tools when it decides to.

Claude Code's actual design: `CLAUDE.md` files are *"naively dropped into context up front,"* while `glob` and `grep` provide JIT access to everything else. The pre-loaded anchor gives the agent enough context to know what's available; JIT gives it the actual data when needed.

The decision boundary depends on the task. For dynamic, exploratory work (research, coding), bias toward JIT. For more static, predictable work (legal contract review, summarization of a known document), bias toward pre-loaded. As Anthropic put it: *"Smarter models require less prescriptive engineering. Do the simplest thing that works."*

## Variants & related patterns

- [**Progressive disclosure**](progressive-disclosure.md) — layered JIT: metadata first, then summary, then full content.
- [**Structured note-taking**](structured-note-taking.md) — externalized state, accessed via the same JIT tools.
- [**Compaction**](compaction.md) — JIT complements compaction: less in context means compaction is needed less often.
- [**Code Mode / Code execution with MCP**](#) — a specific implementation of JIT for MCP tools: agent writes code that fetches data instead of pre-loading.
- **Agentic RAG** — RAG with a JIT-style loop (decide → retrieve → decide → re-retrieve → answer).
- **MCP server design** — well-designed MCP servers expose JIT-friendly tools (paginated, filterable, scoped queries) rather than dump-everything endpoints.

## When NOT to use

- **One-shot, well-scoped tasks.** Single question, single retrieval — just pre-load. Adding a JIT loop adds latency.
- **When latency dominates.** Each JIT tool call is an extra LLM round-trip (plus the tool execution). For sub-second user-facing interactions, pre-loaded RAG with reranking is often faster.
- **When the model is weak at tool use.** Smaller / older models may waste calls fetching wrong files, looping on dead ends, or failing to identify what's relevant. Pre-loading is a safer default.
- **When the data is small enough.** If your entire corpus comfortably fits in 30% of the context window, JIT is overkill — just load it.
- **For audit-critical workflows** where you need to prove the agent considered specific evidence — pre-loading makes that explicit; JIT is harder to audit.
- **When the cost of an empty context is too high** — e.g., compliance or safety contexts where the agent must *always* have certain constraints in front of it. Pre-load those, JIT everything else.

## Implementations

| Tool / framework | JIT support |
|---|---|
| **Claude Code** | The reference implementation. `glob`, `grep`, `read_file`, `Bash` as JIT primitives. |
| **Anthropic Agent SDK** | Provides the same primitives to your custom agents. |
| **OpenAI Codex CLI / Codex app** | Equivalent file/grep tools; same JIT pattern. |
| **MCP filesystem servers** | Universal JIT file access for any MCP-compatible client. |
| **MCP database servers** | JIT queries against SQL / NoSQL. |
| **LangGraph + tool nodes** | DIY: any tool that returns specific data instead of dumping a corpus. |
| **LlamaIndex query engines** | A retriever per data source; the agent selects which to call. |
| **Pydantic AI** | Tool-based architecture is inherently JIT-friendly. |
| **Aider, Cline, Cursor Agent** | All use grep-and-read JIT for code. |

## Companies using JIT context

- **Anthropic** ✅ — Claude Code is the canonical example. ([source](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents))
- **OpenAI** ✅ — Codex CLI/app explicitly designed for filesystem JIT. The Codex announcement notes file/grep tools as core primitives.
- **Cursor, Aider, Cline, OpenCode** ✅ — all major coding agents use this pattern; visible in their source.
- **Sourcegraph (Cody)** ✅ — built around code-search-as-retrieval (which is JIT by design).
- **Perplexity, ChatGPT search, Claude Web Search** ⚠ — search results are JIT vs pre-loaded; specific architecture details vary.
- **Devin (Cognition)** ⚠ — long-horizon coding necessarily uses JIT; specifics not public.

## Further reading

- [Effective context engineering — "Context retrieval and agentic search" section](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic, Sept 2025 (canonical source)
- [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — Anthropic, Nov 2025 (JIT applied to MCP tools — 98.7% token reduction)
- [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) — the reference for JIT in coding agents
- [MCP server design guidelines](https://modelcontextprotocol.io/) — designing tools for agentic JIT use

---

*Diagram source: [`../diagrams/src/jit-vs-preload.d2`](../diagrams/src/jit-vs-preload.d2), [`../diagrams/src/hybrid-context.d2`](../diagrams/src/hybrid-context.d2)*
