# Compaction

**Aliases:** context summarization, context window compaction, conversation distillation
**Category:** Context Engineering
**Sources:**
[Anthropic — Effective context engineering for AI agents (Sept 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
[Claude Code internals (referenced in same post)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
[Anthropic — Tool result clearing (Claude Developer Platform feature, 2025)](https://docs.claude.com/en/docs/build-with-claude/memory-tool)

---

## Problem

> [!TIP]
> **ELI5.** You've been working with your assistant for hours. The notebook is almost full. If you keep going, the next page won't fit. The naive options are: (1) throw the notebook away and start over (you lose everything), or (2) keep writing in tinier handwriting until it's illegible (you keep everything but no one can read it). Compaction is the third option: pause, write a *summary* of what's mattered so far on page one of a new notebook, then keep going from there.

A long-horizon agent — a coding agent working on a multi-hour refactor, a research agent that's done 40 tool calls, a customer-service agent on its 200th turn — eventually runs out of context window. Three things tend to happen at that point, none good:

- **Hard truncation**: drop the oldest messages. Cheap, but the agent forgets the original task, early decisions, and discovered facts. Quality collapses.
- **Sliding window**: keep only the last N tokens. Same problem with a smoother edge.
- **Refuse to continue**: hit the limit, error out, lose all state. Common in 2023-era frameworks.

The Anthropic recommendation (and the de facto pattern in production agent harnesses by late 2025): **don't let the window fill at all**. When you get close to the limit, **summarize the conversation into a compact, high-fidelity distillation and restart the context with just that summary plus the most-recently-relevant raw data**. The agent continues as if the summary *is* its memory of what came before — because for context-window purposes, it now is.

Compaction is the first lever you reach for in long-horizon agents. Anthropic's wording: "Compaction typically serves as the first lever in context engineering to drive better long-term coherence."

## How it works

> [!TIP]
> **ELI5.** When the context is about to overflow, you (the harness, not the user) call the LLM with a special prompt: "Here's everything we've been doing. Write a summary that captures the architectural decisions, unresolved bugs, in-progress work, and any constraints — but drop the chatter and the duplicated tool outputs." Then you replace the giant message history with that summary plus the last few things, and the agent continues.

![Compaction flow — before, summarize, after](../diagrams/svg/compaction-flow.svg)

The mechanics, as implemented in Claude Code and similar agent harnesses, work like this. The harness watches the running token count of the agent's context window. When that count approaches some threshold — typically 75-85% of the model's context limit — the harness triggers a compaction step.

In the compaction step, the harness sends the **current message history** to the model (or a smaller summarizer model) with a **dedicated compaction prompt**. The compaction prompt explicitly tells the model what to preserve and what to drop. Anthropic's wording, paraphrased from the Sept 2025 post: *"Preserve architectural decisions, unresolved bugs, implementation details that affect future work, and active constraints. Discard redundant tool outputs, resolved sub-tasks, duplicate retrievals, and conversational pleasantries."*

The model produces a **summary** — typically 1-5K tokens distilling tens or hundreds of thousands of tokens of conversation. The harness then **restarts the context** with: system prompt + tool definitions + summary + the **5 most-recently-accessed files** (Claude Code's specific design choice — the "active working set" of the current sub-task). Total: usually under 20K tokens, with hundreds of thousands of tokens of breathing room ahead.

From the agent's perspective, this is transparent. It doesn't see the compaction step. It just continues from a (much smaller) context, with the summary serving as its working memory of what came before.

### The art is in the summarization prompt

Compaction is high-leverage but also high-risk. Aggressive compaction can drop subtle but critical context — a constraint mentioned once 50 turns ago, a bug that was discovered but not yet fixed, a decision that was made but not yet implemented. Anthropic's specific guidance:

![Compaction trade-off](../diagrams/svg/compaction-tradeoff.svg)

The recommended development process for a compaction prompt: **maximize recall first, then iterate on precision**. Start with a prompt that captures every relevant piece of information from real failure traces (even at the cost of bloated summaries). Then, once recall is solid, tighten the prompt to eliminate superfluous content. The opposite order — start with a tight summary, then add detail back when things break — is much harder because you don't know what you're missing until the agent fails downstream.

Test compaction prompts on **complex real agent traces**, not synthetic conversations. The failure modes — subtle decisions silently dropped, constraints forgotten, "I already tried that" facts lost — only surface when the agent continues into the next phase of work.

### Tool result clearing — the lightest form of compaction

The safest, lightest-touch form of compaction is **tool result clearing**: once a tool has been called and its result consumed for the next decision, the *raw* tool output gets dropped from history while the *fact that the tool was called* (and a one-line summary of what it found) stays. This is now a managed feature on the Claude Developer Platform.

Example: an agent calls `read_file("config.py")` and gets back 2,000 lines. After it uses that content to make its next decision, the harness replaces the raw 2,000 lines with `[tool: read_file("config.py") — 2000 lines, see structured_notes_id=42]`. The full content stays available via the notes/memory tool if needed again, but doesn't keep eating context every turn.

Tool result clearing alone can extend an agent's effective horizon by 5-10× before full compaction becomes necessary.

### When the summary itself becomes the bottleneck

After enough compactions, the summary itself contains summaries of summaries of summaries — and signal degrades. Two mitigations:

1. **Structured note-taking** in parallel: while compaction happens in the context window, the agent also writes structured notes to files (`NOTES.md`, `STATE.json`) via tools. Those notes are *not* compacted and remain authoritative. The compaction summary becomes "for orientation," not the source of truth. See [`structured-note-taking.md`](structured-note-taking.md).
2. **Sub-agent spawning**: when the current agent's context has been compacted multiple times, the lead agent spawns a *fresh* sub-agent with a clean context, a focused task description, and references (file paths) to the persistent notes. The sub-agent does its work, returns a summary, and the lead picks up. See multi-agent patterns in [`../agt/`](#).

## Variants & related patterns

- [**Tool result clearing**](#) — the lightest-touch compaction; clear raw results once consumed.
- [**Structured note-taking**](structured-note-taking.md) — the durable companion to compaction; what the summary references.
- [**Sub-agent architectures**](#) — the heavy-hammer alternative: instead of compacting, spawn fresh contexts.
- [**Just-in-time context**](just-in-time-context.md) — avoid needing compaction by not pre-loading.
- [**Context rot**](context-rot.md) — the underlying phenomenon compaction mitigates.
- **Claude Code rainbow deployments** — operational pattern that interacts with compaction (mid-deploy you may need to handle agents at any compaction stage).
- **OpenAI's "summarize_history" tools** in Assistants API — a managed equivalent.

## When NOT to use

- **Short agents.** If the agent comfortably finishes within 30% of the context window, don't add compaction. The summarization step costs a full LLM call and adds latency.
- **Compliance-critical agents** where every prior message must be preserved verbatim (legal discovery, audit trails). For those, log to external storage and use just-in-time retrieval rather than in-context history.
- **When tool result clearing alone suffices.** Try this first. It's cheaper and lossless for tool outputs you don't re-reference.
- **When the model is reasoning-heavy.** Reasoning models (o1, R1, Claude extended-thinking) generate a lot of thinking tokens that don't need to persist; native thinking-token discarding handles this case better than compaction.
- **For very narrow, deterministic agents** — extract-then-act flows, single-step automations. Adding compaction infrastructure is overkill.

## Implementations

| Tool / harness | Compaction support |
|---|---|
| **Claude Code** | Built-in. Triggers near context limit. Preserves architectural decisions + active files. |
| **Claude Developer Platform** | **Tool result clearing** as a managed feature; full compaction via memory tool patterns. |
| **OpenAI Assistants API** | `truncation_strategy` parameter: `auto` or `last_messages`. Less sophisticated than full compaction. |
| **LangGraph** | DIY: state machine + `summarize` node before context limit. |
| **AutoGen** | Built-in `summarize_message_history` agent. |
| **CrewAI** | Built-in memory compaction. |
| **Letta (formerly MemGPT)** | Compaction is foundational — the framework was built around it. Implements **recursive summarization** explicitly. |
| **Pydantic AI** | DIY via context management. |
| **Roll your own** | Most production teams: a `should_compact()` check before every turn plus a `compact(history) -> summary` LLM call. |

## Companies using compaction

- **Anthropic** ✅ — Claude Code's compaction is the reference implementation. ([source](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents))
- **OpenAI** ⚠ — Assistants API has a managed truncation strategy; not as sophisticated as Anthropic's described approach.
- **Letta (MemGPT)** ✅ — the academic project that introduced the recursive-summarization architecture in late 2023; productionized as Letta.
- **Cursor, Aider, Cline, OpenCode** ⚠ — all coding agents need this; specific implementations not all documented.
- **Devin (Cognition)** ⚠ — multi-hour autonomous coding necessarily uses compaction-like patterns; specific implementation not public.

## Further reading

- [Effective context engineering for AI agents — "Compaction" section](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic, Sept 2025 (the source)
- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) — Packer et al. 2023 (precursor)
- [Letta documentation — memory blocks and recall](https://docs.letta.com/) — the productionized version
- [Tool result clearing](https://docs.claude.com/en/api/messages) — Claude Developer Platform managed feature
- [Claude Code best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices) — Apr 2025

---

*Diagram source: [`../diagrams/src/compaction-flow.d2`](../diagrams/src/compaction-flow.d2), [`../diagrams/src/compaction-tradeoff.d2`](../diagrams/src/compaction-tradeoff.d2)*
