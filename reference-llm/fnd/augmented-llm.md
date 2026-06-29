# The Augmented LLM

**Aliases:** the LLM atom, the agentic primitive, LLM+tools+retrieval+memory, augmented language model
**Category:** Foundations
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[Mialon et al. — Augmented Language Models: A Survey (2023)](https://arxiv.org/abs/2302.07842) ·
[Anthropic — Effective context engineering (Sept 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
[OpenAI function calling docs (2023-2026)](https://platform.openai.com/docs/guides/function-calling)

---

## Problem

> [!TIP]
> **ELI5.** A bare LLM is a stateless text-completion engine — give it words, get words back. To do anything *useful in the world*, it needs three things: **retrieval** (to read knowledge it wasn't trained on), **tools** (to take actions and read fresh data), and **memory** (to carry state across turns). An LLM with any/all of these wired in is an **augmented LLM**. Anthropic calls this the *atomic unit* of agentic systems — workflows compose augmented LLMs in fixed paths; agents drive their own loop over augmented LLMs. Every modern AI product is built on this one primitive.

When LLMs first launched as chat products (ChatGPT, late 2022), they were essentially raw — text in, text out, with whatever knowledge happened to be in their training data, and no ability to act. Within a year, every serious product had bolted on the same three additions: a way to **read fresh context** (retrieval / RAG), a way to **call tools** (function calling), and a way to **remember things across turns** (memory).

In December 2024, Anthropic's [Building effective agents](https://www.anthropic.com/research/building-effective-agents) post crystallized this into a precise concept: the **augmented LLM** is the atomic building block. Workflows and agents alike are just different ways of *composing* augmented LLMs. Understand this primitive and the rest of the agentic-pattern landscape becomes much clearer — every pattern is some arrangement of augmented LLMs talking to each other and to the world.

This page documents the primitive itself. The rest of `reference-llm/` is about how to compose it.

## How it works

> [!TIP]
> **ELI5.** Start with an LLM. Add (one or all of): **retrieval** so it can look stuff up at inference time, **tools** so it can do things in the world and read fresh data, **memory** so it remembers across turns. Each addition is optional and independently composable. Most production deployments have at least two of three.

![The augmented LLM](../diagrams/svg/augmented-llm.svg)

### The three augmentations

**Retrieval.** At inference time, the LLM looks up relevant information from an external store (vector DB, search index, document set, code repo, etc.) and includes it in its context. Examples:
- RAG against a knowledge base.
- Code search before answering a code question.
- Pulling current weather, stock prices, news.
- Fetching the relevant section of a long document.

Retrieval gives the LLM **knowledge it didn't have at training time** — including data that postdates the model's training cutoff, or private/proprietary data the model never saw.

**Tools.** The LLM emits structured calls that the harness executes; results are fed back into the next inference step. Examples:
- Web search, calculator, code execution.
- API calls (Stripe, GitHub, internal systems).
- Filesystem operations.
- Database queries.
- External-service actions (send email, post message, deploy service).

Tools give the LLM the ability to **act in the world** and to **read live data** that isn't available via static retrieval. The boundary between "retrieval" and "tools" is fuzzy — a search tool *is* retrieval, just dynamic.

**Memory.** State that persists across LLM calls within a session, or across sessions. Examples:
- Conversation history within a chat session.
- Per-user facts ("the user prefers Python over JavaScript").
- Per-project context ("we use snake_case in this repo").
- Long-term episodic memory (past interactions the agent can recall).
- Working notes (the [structured note-taking](../ctx/structured-note-taking.md) pattern).

Memory gives the LLM **continuity** — without it, every call is amnesiac.

### Why this is the right unit of composition

The augmented LLM is the right primitive because:

1. **Every useful LLM product has at least one of the three.** A pure-completion LLM is a research demo, not a product.
2. **They compose orthogonally.** You can add tools without retrieval, retrieval without memory, etc. Each is independently useful.
3. **It's the unit higher-level patterns operate on.** Workflows orchestrate multiple augmented LLMs in fixed paths. Agents drive a loop over a single augmented LLM. Multi-agent systems coordinate several. The augmented LLM is the noun in all these patterns.
4. **It matches how production code is structured.** Most LLM SDK wrappers (LangChain agents, OpenAI Assistants, Anthropic SDK with tools, Vercel AI SDK) operate at the augmented-LLM level — you configure tools, retrieval, memory once, then call.

### What goes inside vs outside the augmented LLM

Anthropic's framing is precise: the augmented LLM **does one inference call** when invoked. The augmentations happen *around* the LLM, not inside the model weights:

- **Retrieval happens before** the LLM call (build the context).
- **Tools execute after** the LLM emits a call (run the tool, feed result back, possibly trigger another call).
- **Memory wraps around** the LLM call (read state in, write state out).

The model itself is unchanged. The augmentation is *system-level* — the harness/framework code around the LLM call. This is what makes the same model (Claude, GPT, Gemini, Llama) work in dramatically different products: same atom, different augmentation arrangements.

### Augmented vs reasoning LLMs

A subtle 2024-2026 development: **reasoning models** (OpenAI o1/o3/o5, Claude with extended thinking, Gemini 2.5 with thinking, DeepSeek R1, Qwen QwQ) have *internalized* part of what augmentation used to provide. They reason at length before answering, which was previously done via [reasoning prompts](reasoning-prompts.md) and external orchestration.

But they're still augmented LLMs in Anthropic's sense — they still need retrieval, tools, and memory at the system level. The "thinking" is internal; the augmentation is external. A reasoning model with tools is still composed exactly the same way.

### Variants and shapes

**Tools-only augmented LLM.** Common for action-taking products (Stripe agent, GitHub agent). Tools are scoped to the product's domain.

**Retrieval-only augmented LLM.** Pure RAG products (Perplexity, internal knowledge-base assistants). No actions, just informed answers.

**Memory-only augmented LLM.** Chat companions with persistent personality and history (Character.AI, Replika, generative agents).

**All three.** Most modern coding agents and assistants. Claude Code, Cursor Agent, Devin, ChatGPT with browse + actions + memory — all are full augmented LLMs.

**Tiny / hosted augmented LLM.** Increasingly common: OpenAI Assistants, Anthropic via SDK with tools+files, Vercel AI SDK — the platform provides the augmentation primitives so you just configure them.

### The cost of augmentation

Augmentation isn't free. Each augmentation adds:
- **Context tokens** — retrieved chunks take space; memory takes space; tool definitions and results take space. Risk of [context rot](../ctx/context-rot.md).
- **Latency** — retrieval round-trips, tool calls add seconds.
- **Cost** — every augmentation typically increases tokens per inference, and tool calls add their own costs.
- **Failure modes** — tools fail, retrieval returns irrelevant results, memory becomes stale.

Choosing *which* augmentations matter for a given product is design work. Not every product needs all three.

### Composition into higher-level patterns

Once you have augmented LLMs, you can compose them:

- **Workflow patterns** ([workflows vs agents](../agt/workflows-vs-agents.md)): wire several augmented LLMs together in a fixed graph. Prompt chaining, routing, parallel sectioning, etc.
- **Single-agent loop** ([single-agent-with-tools](../agt/single-agent-with-tools.md)): give one augmented LLM control over its own loop.
- **Multi-agent** ([multi-agent-orchestration](../agt/multi-agent-orchestration.md), [sub-agent architectures](../agt/sub-agent-architectures.md)): multiple augmented LLMs cooperating.

Every more complex pattern in this reference reduces to "arrange augmented LLMs into a particular topology."

### The 2026 baseline expectation

By 2026, when someone says "build an LLM-powered feature," the default assumption is that you'll build *at least* an augmented LLM — not a raw LLM. A raw LLM is essentially a research toy; production starts at "LLM + at least one of {retrieval, tools, memory}." This is true across SDKs (OpenAI Assistants, Anthropic, Gemini), frameworks (LangChain, LlamaIndex, Vercel AI SDK, Mastra), and hosted platforms (Bedrock, Vertex, Foundry).

The 2024 pattern was "build a bare chatbot, then add features." The 2026 pattern is "start with augmentation; the bare chatbot was the special case."

## Variants & related patterns

- [**Workflows vs agents**](../agt/workflows-vs-agents.md) — the two ways to compose augmented LLMs.
- [**Single-agent with tools**](../agt/single-agent-with-tools.md) — the simplest non-trivial composition.
- [**ReAct**](react.md) — the prompting pattern that drives the tool-using augmented LLM.
- [**Structured outputs / function calling**](structured-outputs.md) — how tools are typed in practice.
- [**Reasoning prompts**](reasoning-prompts.md) — how to make the LLM core reason better.
- [**Context engineering overview**](../ctx/context-engineering-overview.md) — disciplined management of what goes *into* the augmented LLM.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — retrieval pattern when context budget is tight.
- [**Compaction**](../ctx/compaction.md) — memory pattern when conversation history grows.
- **RAG** — the canonical retrieval pattern (covered in `ret/`).
- **Tool use / function calling** — the canonical tools pattern.

## When NOT to use augmentation

- **Pure completion tasks** with no need for external knowledge, no actions, no history — a bare LLM is sufficient. Increasingly rare in production.
- **Latency-critical paths** where each augmentation adds milliseconds — you may want to skip optional augmentations.
- **High-throughput inference at scale** where tools/retrieval are too slow — fall back to fine-tuning or precomputed augmentation.

For interactive products with users, augmentation is essentially always worthwhile.

## Implementations

| SDK / framework | First-class support for ... |
|---|---|
| **OpenAI Assistants API** | Tools, retrieval (file_search), memory (threads) — all three |
| **Anthropic Claude SDK** | Tools, retrieval (via context), MCP integration |
| **Vercel AI SDK** | Tools, retrieval helpers, history utilities |
| **LangChain / LangGraph** | All three; emphasizes graph composition |
| **LlamaIndex** | Retrieval-first, tool support |
| **Mastra** | Modern TS-first agentic framework with all three |
| **Mirascope, Instructor** | Lightweight wrappers over function calling |
| **OpenAI Agents SDK (2025)** | Native augmented-LLM as agent primitive |
| **Anthropic Workbench** | Configure all three in console |
| **Google Vertex AI Agent Builder** | Hosted augmented-LLM primitives |
| **AWS Bedrock Agents** | Hosted augmented-LLM with action groups + knowledge bases |
| **Azure AI Foundry Agents** | Same |

## Companies / products structured as augmented LLMs

Practically all production LLM products. A representative sample:

- **Anthropic** ✅ — Claude itself, Claude Code, Workbench all built on this primitive ([source](https://www.anthropic.com/research/building-effective-agents)).
- **OpenAI** ✅ — ChatGPT (with browsing, tools, memory), Assistants API, Operator, Codex.
- **Google** ✅ — Gemini products with grounding, function calling, conversation memory.
- **Perplexity, Phind, You.com** ✅ — retrieval-augmented LLMs.
- **GitHub Copilot, Cursor, Aider, Cline** ✅ — tool-augmented LLMs.
- **Devin, Codex, Replit Agent** ✅ — all three augmentations.
- **Character.AI, Replika** ✅ — memory-augmented LLMs.
- **Notion AI, Linear AI, Stripe AI** ✅ — all three.
- **Every enterprise RAG deployment** ✅ — retrieval-augmented at minimum.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024 (canonical framing)
- [Augmented Language Models: A Survey](https://arxiv.org/abs/2302.07842) — Mialon et al. 2023 (academic survey)
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic Sept 2025
- [OpenAI function calling guide](https://platform.openai.com/docs/guides/function-calling)
- [Anthropic tool use docs](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use)
- [The future of AI is agentic](https://huyenchip.com/2025/01/07/agents.html) — Chip Huyen

---

*Diagram source: [`../diagrams/src/augmented-llm.d2`](../diagrams/src/augmented-llm.d2)*
