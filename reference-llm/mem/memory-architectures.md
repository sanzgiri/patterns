# Memory Architectures for LLM Agents

**Aliases:** agent memory, MemGPT/Letta architecture, working/episodic/semantic/procedural memory, hierarchical memory, conversation memory
**Category:** Memory
**Sources:**
[Packer et al. — MemGPT: Towards LLMs as Operating Systems (Oct 2023)](https://arxiv.org/abs/2310.08560) (became [Letta](https://github.com/letta-ai/letta)) ·
[Park et al. — Generative Agents (Apr 2023)](https://arxiv.org/abs/2304.03442) (memory streams) ·
[Weng — LLM Powered Autonomous Agents (June 2023)](https://lilianweng.github.io/posts/2023-06-23-agent/) (canonical memory taxonomy) ·
[Anthropic — Memory and context management](https://www.anthropic.com/news/projects) (Claude Projects) ·
[OpenAI — Memory in ChatGPT (Feb 2024)](https://openai.com/index/memory-and-new-controls-for-chatgpt/)

---

## Problem

> [!TIP]
> **ELI5.** LLMs are stateless. Every call is a fresh start — no memory of yesterday's conversation, no idea what you like, no record of what they tried last time. "Memory" in LLM agents is whatever you keep around *outside* the model and inject back into the prompt at the right moment. The trick is that there's no single "memory" — there are *kinds* of memory that work very differently. Borrowed loosely from cognitive science, modern agents distinguish **working memory** (the current context window), **episodic memory** (past conversations), **semantic memory** (facts about the world or user), **procedural memory** (skills and routines), and **archival memory** (everything else, retrievable on demand). Production agents use all five, each backed by different storage and retrieved differently.

The LLM is a function: `input → output`. No state persists across calls. Any "memory" the agent appears to have is engineered by the harness around it — selectively saving information and selectively re-injecting it into the next prompt.

The challenge is that what to save, where to save it, and when to inject it are all different problems. A user preference ("metric units, please") needs different treatment than "what did we discuss in our March 4 session" which needs different treatment than "how to invoke the refund API."

The dominant taxonomy comes from Lilian Weng's [June 2023 post](https://lilianweng.github.io/posts/2023-06-23-agent/) (LLM components: planning, memory, tools), refined by Charles Packer's [MemGPT/Letta paper](https://arxiv.org/abs/2310.08560) (Oct 2023) which made the analogy to operating-system memory hierarchies explicit. By 2024-2026 this taxonomy is the de-facto standard, used in Letta, LangGraph memory, ChatGPT memory, Claude Projects, and most production agent frameworks.

## How it works

> [!TIP]
> **ELI5.** Five layers, from "fast and in your face" to "slow and on demand." Working memory is the context window itself — only thing the LLM sees directly each turn. Everything else gets *loaded into* working memory when relevant. Episodic memory remembers conversations; you retrieve by similarity. Semantic memory remembers facts about the world or user; you query by key. Procedural memory is "how to do X" — Skills, system prompts, tool configs. Archival memory is the long tail — docs, code, emails — retrieved via RAG. Each layer has different storage, different access patterns, different update rules.

![Memory architectures](../diagrams/svg/memory-architectures.svg)

### Working memory (the context window)

This is what the model sees *this turn*. It includes:
- System prompt
- Tool descriptions
- Conversation history (or summary)
- Retrieved chunks ([RAG](../ret/basic-rag.md))
- Tool results
- User's latest message

Working memory is the only "memory" the LLM has direct access to. Everything below this line is engineered injection.

Size: bound by context window (200K–2M tokens in 2026 frontier models). But [context rot](../ctx/context-rot.md) means *effective* working memory is often much smaller — maybe 30-50K tokens before quality degrades on most models. Working memory is also expensive (every token costs at inference time), so even when capacity allows, you don't fill it indiscriminately.

Persistence: dies at conversation end. Working memory is *not* persistent storage.

### Episodic memory (past interactions)

Stores past conversations or events as discrete units ("episodes"). Typically:
- Each conversation (or each turn) is stored as a chunk
- Embedded for semantic similarity retrieval
- Indexed with metadata (timestamp, user, topic)

Retrieved by: similarity to current query, or by time range, or by user/session ID.

**Implementations:**
- **Letta (MemGPT)**: episodic memory is a "recall" buffer; older episodes get summarized and archived
- **LangGraph long-term memory**: per-thread episodic store; retrieved into working memory
- **OpenAI ChatGPT memory**: implicit episodic store; ChatGPT silently retrieves relevant past chats
- **Anthropic Projects**: persistent project context across conversations

The pattern Generative Agents popularized (Park et al. Apr 2023): a "memory stream" of timestamped observations, retrieved by combining recency × importance × similarity. Recency matters because it weights recent events higher; importance is an LLM-assigned score; similarity is vector distance to current query.

### Semantic memory (facts)

Structured knowledge about the user or world:
- "User prefers Python over JS"
- "User's company is ACME Corp"
- "User has a pet golden retriever named Rex"

Unlike episodic memory (which stores *events*), semantic memory stores *facts*. Facts get **updated**, not just appended: if the user moves cities, old fact gets replaced, not added alongside.

**Implementations:**
- **Letta**: "core memory" — a small, always-in-context fact store, user-modifiable by the agent itself
- **ChatGPT memory**: user-visible fact list ("Things ChatGPT remembers about you")
- **mem0** (open-source library): dedicated semantic memory layer
- **Custom key-value stores** with LLM-driven extraction (the agent says "remember that user X prefers Y" → update)

Key engineering decision: **who writes to semantic memory?** Three patterns:
1. **User-driven**: user explicitly tells the agent ("remember I'm vegetarian")
2. **Agent-driven**: agent infers facts from conversation, writes them silently
3. **Hybrid**: agent proposes, user confirms

Production systems use all three. Pure agent-driven has high accuracy risk (false facts pollute the store); pure user-driven misses obvious updates.

### Procedural memory (how to do things)

Stored skills, workflows, tool configurations:
- A `SKILL.md` file describing how to handle a kind of task
- Customized system prompts per task type
- Tool definitions and configurations
- Few-shot examples for specific tasks
- Learned macros or routines

Procedural memory is what makes an agent reusable across sessions without re-explaining. The [Skills](../skills/skill-md-format.md) ecosystem (Claude Code skills, pi skills, etc.) formalizes this layer.

Procedural memory is typically *human-authored* (or human-reviewed if agent-generated). It's the slowest-moving memory layer and has the highest lifetime value per write.

### Archival memory (everything else)

The long-tail store:
- Documents, PDFs, HTML
- Code repositories
- Emails, Slack history
- Database records
- Past project artifacts

Retrieved via RAG ([basic-rag](../ret/basic-rag.md), [advanced-rag](../ret/advanced-rag.md), [contextual-retrieval](../ret/contextual-retrieval.md)). This is the largest layer by far — potentially terabytes — and the cheapest per byte to store.

Archival memory is what most "Chat with your docs" products consist of. The other four layers are mostly absent in those simpler products; they emerge as the product matures into a true agent.

### The MemGPT/Letta architectural insight

The MemGPT paper made memory hierarchy explicit and gave the model **tools to manipulate its own memory**: the LLM can call tools like `memory_append`, `memory_replace`, `archival_search`, `recall_search`. This treats memory like a programmer treats RAM and disk — explicit, controllable, debuggable.

The "memory tools" pattern is now in:
- Letta (the successor to MemGPT)
- LangGraph memory (similar tools as primitives)
- OpenAI Assistants memory (more implicit)
- Custom agent frameworks

The alternative (used in ChatGPT memory, Claude Projects) is **implicit memory**: the harness manages memory; the LLM doesn't directly see the operations. Simpler for users; less debuggable.

### How memory is loaded into working memory

At each turn:
1. **Always-loaded**: system prompt, core semantic facts (small, hot)
2. **On-demand**: retrieve episodic memory by similarity to current input
3. **On-demand**: retrieve archival memory via RAG when query suggests doc lookup
4. **Conditional**: load relevant procedural memory (skills) based on task classification
5. **Compressed**: long conversation history may be replaced by a [compaction](../ctx/compaction.md) summary

The total injected into working memory must fit within context budget. Trade-offs:
- Too little injection → agent forgets relevant context
- Too much injection → [context rot](../ctx/context-rot.md), increased cost

This is fundamentally [context engineering](../ctx/context-engineering-overview.md). Memory architecture is the *storage* side; context engineering is the *what-to-load* side.

### Anti-patterns

- **One vector store for everything.** Mixing episodic, semantic, archival into one undifferentiated store makes retrieval noisy and updates hard.
- **No compaction.** Working memory grows unboundedly until context window fills.
- **Pure agent-driven semantic memory** without verification — accumulates wrong facts.
- **Conversation history as the only memory layer.** Works for chatbots; fails for agents.
- **Skipping procedural memory** (skills) — every session has to re-establish how to do things.
- **No memory eval set.** You can't tell if memory changes help.
- **Per-user memory leaked across users.** A common security bug in multi-tenant systems.
- **Treating memory as cache** without invalidation — stale facts persist.

### Production engineering

- **Per-user isolation**: critical for privacy. Memory is per-tenant by default.
- **Provenance**: every memory write should record source (which conversation? which message?).
- **TTL / expiry**: some facts should age out; episodic memory often pruned by time.
- **Summary vs full**: keep both a compressed summary and the raw episode when budget allows.
- **Retrieval eval**: measure recall@K for memory retrieval just like for RAG.
- **Memory drift detection**: when semantic facts contradict, surface for human review.
- **Backup**: memory loss is a worse UX than no memory.
- **Audit log**: who/what wrote each memory entry, when.

## Variants & related patterns

- [**Compaction**](../ctx/compaction.md) — working-memory pruning.
- [**Structured note-taking**](../ctx/structured-note-taking.md) — agent writes notes to persist info.
- [**Context engineering overview**](../ctx/context-engineering-overview.md) — what to load when.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — load only when needed.
- [**Prompt caching**](prompt-caching.md) — reduces cost of fat working memory.
- [**Long-context vs RAG**](long-context-vs-rag.md) — when memory needs to scale.
- [**Basic RAG**](../ret/basic-rag.md) / [**advanced RAG**](../ret/advanced-rag.md) — archival memory.
- [**SKILL.md format**](../skills/skill-md-format.md) — procedural memory.
- **Generative Agents memory stream** (Park et al. 2023).
- **Operating-system memory hierarchy** — the inspiration.

## When NOT to use full memory architecture

- **Single-turn agents** (one-shot Q&A): just working memory, no persistence needed.
- **Stateless workflow nodes** (e.g., a translation step): keep them pure.
- **Compliance-sensitive deployments** where retaining anything is restricted.
- **Tiny apps** where the engineering complexity exceeds the value.

## Implementations

| Tool | Memory layers supported |
|---|---|
| **Letta (formerly MemGPT)** | Working, recall (episodic), core (semantic), archival |
| **LangGraph** | All five via custom stores; built-in checkpointing |
| **mem0** | Semantic + episodic (open-source) |
| **OpenAI Assistants API** | Conversation state + file search (archival); memory beta |
| **OpenAI ChatGPT memory** | Semantic + implicit episodic |
| **Anthropic Claude Projects** | Procedural + archival (project files); session-level working |
| **Custom agent harnesses** | Build with vector DB + KV store + RAG |
| **LangChain Memory** | Conversation memory (working) + episodic buffer + summary |
| **LlamaIndex memory** | Conversation + entity (semantic) + summary |

## Companies / products with memory architectures

- **OpenAI ChatGPT** ✅ — explicit memory feature (Feb 2024); cross-conversation episodic + semantic.
- **Anthropic Claude (Projects)** ✅ — procedural + archival per-project.
- **Google Gemini (Gems)** ✅ — similar to Projects.
- **Letta** ✅ — productizes the MemGPT architecture.
- **Cursor, Continue, Cody** ⚠ — procedural memory via `.cursorrules` / settings; archival via codebase index.
- **Inflection Pi** ⚠ — persistent personality + memory across sessions.
- **Character.AI** ⚠ — character-level semantic + episodic per user.
- **mem0 customers** ✅ — semantic + episodic memory as a service.
- **Replika, Pi** ⚠ — relationship-grade episodic memory.
- **OpenAI Operator, Anthropic Computer Use** ⚠ — task-level procedural memory.

## Further reading

- [LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/) — Lilian Weng (canonical memory taxonomy)
- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) — Packer et al. 2023
- [Letta GitHub](https://github.com/letta-ai/letta) — the MemGPT successor
- [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442) — Park et al. 2023
- [LangGraph long-term memory docs](https://langchain-ai.github.io/langgraph/concepts/memory/)
- [mem0 GitHub](https://github.com/mem0ai/mem0)
- [ChatGPT memory announcement](https://openai.com/index/memory-and-new-controls-for-chatgpt/) — Feb 2024
- [Cognitive architectures for language agents](https://arxiv.org/abs/2309.02427) — Sumers et al. (CoALA framework, broader theory)

---

*Diagram source: [`../diagrams/src/memory-architectures.d2`](../diagrams/src/memory-architectures.d2)*
