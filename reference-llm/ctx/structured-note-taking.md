# Structured Note-Taking (Agentic Memory)

**Aliases:** agentic memory, externalized scratchpad, persistent notes, file-based memory
**Category:** Context Engineering
**Sources:**
[Anthropic — Effective context engineering for AI agents (Sept 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
[Anthropic Sonnet 4.5 launch — memory tool](https://www.anthropic.com/news/claude-sonnet-4-5) ·
[Claude playing Pokémon — case study](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
[Packer et al. — MemGPT (2023, precursor)](https://arxiv.org/abs/2310.08560)

---

## Problem

> [!TIP]
> **ELI5.** Your brilliant assistant has a 200-page notebook, but you're working on a multi-month project. Every important decision, every constraint, every bug they discovered — if it only lives in the notebook, it gets thrown out when the notebook fills up and they have to start fresh. The solution: give them a *filing cabinet next to the desk*. Decisions, in-progress checklists, discovered facts go in the cabinet. The notebook only needs to hold what they're working on *right now*. When they need to remember "what did we decide about X three weeks ago?", they open the right folder.

A long-running agent accumulates state: decisions it made, constraints it discovered, facts it looked up, sub-tasks it completed, errors it hit and how it recovered. All of this is valuable for *future* decisions. The naive option is to leave it all in the message history — but the message history is the context window, the context window is finite, and [context rot](context-rot.md) degrades the agent's ability to actually use that state as the window grows.

[Compaction](compaction.md) helps by summarizing the history. But compaction is lossy — subtle decisions get smoothed over, "I already tried that" details get dropped, multi-step plans collapse to a sentence. After several compactions, the summary itself becomes the bottleneck: the agent re-discovers facts it already knew, contradicts decisions it already made, and forgets the original goal.

The fix the production agent community converged on in 2025: **don't keep state in the conversation at all**. Keep it in **files** — read and written by the agent via tools, persisted across compactions, across sub-agents, even across full agent restarts. The conversation becomes the *working memory*; the files become the *long-term memory*.

## How it works

> [!TIP]
> **ELI5.** Give the agent a set of files it can write to as it works — TODO.md for the checklist, NOTES.md for discovered facts, STATE.json for decisions, workspace/ for partial outputs. When it needs to look something up, it greps or reads the file. The conversation only contains what's relevant *right now*; the files contain everything that's ever mattered.

![Structured note-taking — externalized memory](../diagrams/svg/structured-note-taking.svg)

The agent runs in its context window as usual, but it has access to **file I/O tools** — read, write, edit, grep, glob, list. As it works, it deliberately writes structured notes to files outside the context:

- **`TODO.md`** — the active checklist of what the agent is working on. Items get marked complete as they finish.
- **`NOTES.md`** (or `LOG.md`) — discovered facts, "I tried X and it failed because Y," surprises, things to remember.
- **`STATE.json`** (or `DECISIONS.md`) — explicit decisions: "we chose architecture A over B because Z." Constraints discovered.
- **`workspace/`** — partial outputs, artifacts being constructed, intermediate files.

When the agent needs to recall prior work, it reads the relevant file with a tool. When [compaction](compaction.md) kicks in, the **context** gets summarized — but the **notes survive**, untouched, on disk. The summary in the compacted context refers to the notes by filename or by structured ID; the agent re-reads them when it needs the full picture.

![Notes survive compaction](../diagrams/svg/notes-survive-compaction.svg)

That second property — **notes survive compaction** — is the killer feature. Compaction is allowed to be lossy because the notes hold the authoritative version. Sub-agent spawning (see [`../agt/`](#)) becomes trivial: the parent passes the notes' file paths to a sub-agent with a fresh context, and the sub-agent loads exactly the notes it needs for its specific task. Agent restarts become safe: a new agent process can pick up by reading `STATE.json` and `TODO.md`.

### The Claude playing Pokémon case study

Anthropic uses Claude-playing-Pokémon as a concrete example in the Sept 2025 post. Without any prompting *about how memory should work*, Claude developed its own notes structure: maps of explored regions, key-achievement tallies, combat strategies that worked against different opponents. It maintained precise multi-thousand-step tallies — *"for the last 1,234 steps I've been training my Pokémon in Route 1, Pikachu has gained 8 levels toward the target of 10."*

After context resets — and Pokémon runs span thousands of game steps, well beyond any context window — the agent reads its own notes and continues multi-hour training sequences or dungeon explorations. This is impossible without externalized memory; it works straightforwardly with it.

### The structure matters

The "structured" in structured note-taking is load-bearing. A pile of free-form journal entries doesn't compose well with the agent's read tools. The patterns that work:

- **Append-only logs** for time-ordered facts. Easy to write, grep-friendly.
- **Section-keyed Markdown** (`## Decisions / ## Constraints / ## Open Questions`). Skimmable by an LLM with `read_file` or `grep`.
- **JSON or YAML state objects** for explicit decisions you'll re-read. Easier to inject into prompts than prose.
- **Checklists with status markers** (`- [x] done`, `- [ ] todo`) — Claude Code's TODO list does this.
- **Folder hierarchies** that match task structure (`research/topic-A/`, `research/topic-B/`). Per Anthropic, *"folder hierarchies, naming conventions, and timestamps all provide important signals."*

The hint about *naming conventions and timestamps* is the practical insight: when the agent reads its notes, **filenames and metadata are part of the context**. `test_utils.py` in `tests/` implies a different purpose than `test_utils.py` in `src/core_logic/`. `2026-06-28-deployment-postmortem.md` is more informative than `notes.md`. Structure your notes' filesystem the way you'd structure a project for a careful human collaborator.

### The 2025 Anthropic memory tool

In late 2025, Anthropic released a **memory tool** in public beta on the Claude Developer Platform as part of the Sonnet 4.5 launch. It's a file-based system specifically designed for this pattern: persistent memory accessed via tool calls, structured around storing and consulting information outside the context window. The release effectively blessed structured note-taking as a *first-class* pattern, not just an emergent practice.

The memory tool is just one implementation. The pattern works with any tool that gives the agent read/write file access — Claude Code's built-in filesystem tools, MCP filesystem servers, custom tools in your harness, even a SQLite database accessed through a tool.

## Variants & related patterns

- [**Compaction**](compaction.md) — the lossy in-context companion; notes give it a safety net.
- [**Just-in-time context**](just-in-time-context.md) — the same philosophy applied to *retrieval*: don't pre-load, fetch on demand.
- [**Progressive disclosure**](progressive-disclosure.md) — also fetch-on-demand, but layered (metadata → summary → full).
- [**Context plumbing & reinforcement**](context-plumbing-reinforcement.md) — feeding state back to the agent at every tool call.
- **Sub-agent architectures** (see [`../agt/`](#)) — externalized notes are the medium for inter-agent communication.
- **MemGPT / Letta** — the academic precursor; recursive summarization + external memory.
- **Generative agents** (Park et al. 2023) — memory stream + retrieval scoring; a related architecture for simulation agents.

## When NOT to use

- **Short-lived agents.** A 3-turn extraction agent doesn't need files. Adding file I/O tools adds tool definitions to every call.
- **Read-only / stateless agents** — single-turn Q&A, classification, translation. No state to externalize.
- **High-security contexts** where writing to disk has audit implications. Use an encrypted/managed memory backend, not arbitrary files.
- **When the agent is in a sandbox with no filesystem.** Some browser-based agents, some serverless setups. Use an in-memory key-value store + tool wrapper instead.
- **When latency dominates** — every file read is an extra round-trip. For sub-second agents in tight loops, prefer in-context state.
- **When you can't trust the agent with file write.** A prompt-injection-vulnerable agent with arbitrary file write is dangerous; see [`../sec/lethal-trifecta.md`](#).

## Implementations

| Tool / framework | Note-taking support |
|---|---|
| **Anthropic memory tool** (Sonnet 4.5+ beta) | First-class managed feature. File-based, on the Claude Developer Platform. |
| **Claude Code** | Built-in: agent freely writes to `CLAUDE.md`, `NOTES.md`, project files. |
| **Letta (formerly MemGPT)** | The framework is built around persistent memory blocks + recall queries. |
| **LangGraph state + checkpoints** | Explicit `State` object persisted between nodes; can be backed by files/DB. |
| **AutoGen Memory module** | Built-in memory abstractions; SQLite or vector-store backed. |
| **CrewAI Memory** | Long-term + short-term + entity memory primitives. |
| **OpenAI Assistants — `files` + Code Interpreter** | Files accessible to the assistant across messages; primary access via the code-execution tool. |
| **MCP filesystem servers** | Standard MCP tools for file read/write — universal, any-client. |
| **Custom: SQLite + tool wrapper** | Common production pattern: structured tables + `query_state` / `update_state` tools. |

## Companies using structured note-taking

- **Anthropic** ✅ — Claude Code, Claude Cowork, the memory tool. Claude-playing-Pokémon as case study. ([source](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents))
- **Cognition (Devin)** ⚠ — long-horizon coding agent; widely reported to use persistent notes; not directly verified.
- **Cursor, Aider, Cline** ⚠ — `.cursorrules`, project memory, and CLAUDE.md-style files are widespread.
- **Letta** ✅ — the framework's entire selling point.
- **Sourcegraph (Cody)** ⚠ — discussed in engineering posts as a pattern in their agent work.
- **OpenAI ChatGPT — Memory feature** ⚠ — user-facing memory; closer to long-term user-preference storage than agentic notes, but the same general pattern.
- **Generative Agents (Park et al. 2023)** ✅ — the original academic demonstration (memory stream).

## Further reading

- [Effective context engineering — "Structured note-taking" section](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic, Sept 2025
- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) — Packer et al. 2023 (precursor)
- [Letta documentation — memory blocks](https://docs.letta.com/) — the productionized version
- [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442) — Park et al. 2023 (memory-stream architecture)
- [Anthropic memory tool docs](https://docs.claude.com/en/docs/build-with-claude/memory-tool) — the 2025 Anthropic primitive
- [Claude Code best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices) — Apr 2025

---

*Diagram source: [`../diagrams/src/structured-note-taking.d2`](../diagrams/src/structured-note-taking.d2), [`../diagrams/src/notes-survive-compaction.d2`](../diagrams/src/notes-survive-compaction.d2)*
