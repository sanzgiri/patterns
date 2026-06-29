# Progressive Disclosure

**Aliases:** layered loading, tiered context, multi-level retrieval, drill-down
**Category:** Context Engineering
**Sources:**
[Anthropic — Effective context engineering for AI agents (Sept 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
[Anthropic — Equipping agents for the real world with Agent Skills (Oct 2025)](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) ·
[Anthropic — Code execution with MCP (Nov 2025)](https://www.anthropic.com/engineering/code-execution-with-mcp) ·
UX design term (Nielsen Norman Group) — applied to LLM context in 2025

---

## Problem

> [!TIP]
> **ELI5.** Imagine a Wikipedia article. The first sentence tells you what it's about. The first paragraph gives you the gist. Each section gives more detail. Each link drills down to a whole new article. Nobody reads the whole thing top-to-bottom — you start at the top and follow the threads that matter. **Progressive disclosure** is doing this for an LLM agent: don't dump everything in front of it; give it a tiny summary first, let it ask for more depth on whatever turns out to matter.

You have a lot of context that *might* be useful to an agent: a library of skills, a thousand MCP tool definitions, a corpus of documents, a hierarchy of knowledge-base articles. The naive options are bad:

- **Pre-load everything**: explodes the context window, triggers [context rot](context-rot.md), wastes tokens on stuff the agent never uses. Anthropic measured **150K tokens of tool definitions** for one example agent — most of which were never relevant per call.
- **Pre-load nothing**: the agent doesn't even know what's available, so it can't decide what to fetch.

[Just-in-time context](just-in-time-context.md) solves part of this — keep references in context, fetch content on demand. **Progressive disclosure** generalizes JIT into a *layered* structure: keep a *tiny* description always-loaded, drill into a *moderate* description when something looks relevant, fetch *full content* only when needed.

It's the same idea as the table of contents → chapter → section → paragraph structure in a well-organized manual. The agent navigates the levels itself, paying token cost only for the depth it actually needs.

## How it works

> [!TIP]
> **ELI5.** Three levels. Level 1: a one-line description of every thing the agent might need (skills, tools, files, documents). Always loaded. Level 2: when the agent decides one is relevant, it loads the *full description* (still just text — maybe a page). Level 3: only if needed, it loads the actual content — code to run, full documentation, attached files. Each level is much bigger than the one above it, but the agent only pays the cost as it drills down.

![Progressive disclosure — three levels](../diagrams/svg/progressive-disclosure.svg)

The pattern decomposes content into **three (or more) layers of detail**:

**Level 1 — Metadata / index.** A *tiny* description per item. Loaded into the system prompt or always-available context. Just enough for the agent to decide *whether to look at this item at all*. Typical: a name + 1-line description. Total size: a few hundred tokens for hundreds of items.

**Level 2 — Body / summary.** The full description, instructions, or summary. Loaded on demand when the agent picks the item. Typical: a Markdown document, a few hundred to a few thousand tokens. Loaded by reading a specific file or calling a specific tool.

**Level 3+ — Referenced files / full content.** Bundled scripts, attached PDFs, full documentation, raw data. Loaded only when the agent decides it needs the specifics. May itself be progressively disclosed (an attached folder with its own index).

Each level is *much larger* than the one above it, but the agent only pays the token cost when it drills down. A library of 100 skills might have:

- **Level 1**: ~3,000 tokens (30 tokens × 100 skills) — always loaded
- **Level 2**: skills are 500-2,000 tokens each — loaded when relevant (1-3 per agent run, typically)
- **Level 3**: bundled files might be 10,000+ tokens each — loaded only when the level-2 SKILL.md instructs the agent to read them

For a typical agent run, total token cost is **~3,000 + 2 × 1,500 + 1 × 8,000 ≈ 14,000 tokens** — versus pre-loading everything which would be **~3,000 + 100 × 1,500 + 100 × 8,000 ≈ 953,000 tokens**. A 68× reduction with full coverage.

### Progressive disclosure shows up in three places

The pattern recurs across three of Anthropic's 2025 publications, which is part of why it now has a name:

**1. Agent Skills.** A skill is a folder with `SKILL.md` (YAML frontmatter: `name`, `description`) plus optional bundled files. Three explicit levels:
- **Level 1**: `name` + `description` (loaded at startup into system prompt)
- **Level 2**: full `SKILL.md` body (loaded when agent picks the skill)
- **Level 3+**: bundled files (`reference.md`, `forms.md`, Python scripts — loaded only when SKILL.md instructs)

The Agent Skills [open standard](https://agentskills.io/specification) (Dec 2025) makes progressive disclosure the *defining* design principle. See [`../skills/`](#) for the full pattern.

**2. Code execution with MCP (Code Mode).** Tools are exposed as a file tree under `./servers/<server>/<tool>.ts`. The agent discovers tools by listing the filesystem (Level 1: file names = available tools). It reads specific tool files when it needs their interfaces (Level 2: full TypeScript signature). The tool implementations themselves stay on disk and are only loaded if the agent reads them as reference (Level 3). Result: 98.7% token reduction in Anthropic's example.

**3. Filesystem JIT (Claude Code).** The directory tree is the index (Level 1: folder/file names + sizes). Reading a directory listing or running `glob` retrieves names without content (Level 2: structured listing). Reading a specific file fetches its actual content (Level 3). Each step's output informs the next decision.

### Why it works

**Indexability.** Every level above the bottom is small enough to be a navigable index. The agent can scan the index quickly and decide where to drill — far cheaper than scanning the same content as full text.

**Composability.** Progressive disclosure composes: a Level 3 document (a skill's bundled folder) can itself be progressively disclosed (an index of files, with on-demand reads). Anthropic explicitly mentions this for Skills: *"The amount of context that can be bundled into a skill is effectively unbounded."*

**Predictable cost.** Level 1 cost is fixed and small. Level 2 cost is bounded by *number of items the agent actually touches per run* — usually a handful, not hundreds. Level 3 cost is bounded by *depth of investigation* in that run.

**Trains agent restraint.** The presence of well-structured Level 1 metadata actively *teaches* the agent to scan-then-pick rather than read-everything. With clear names and 1-line descriptions, models are remarkably good at picking the right item without loading its full content.

### Designing for progressive disclosure

Some principles from Anthropic's writing:

- **Names matter.** A Level-1 item's name does a lot of work. `pdf_form_filler` is more useful than `tool_42`.
- **Descriptions are reading material.** The 1-line description is what the agent reads to decide. Write it like a help-string: imperative, specific, distinct from neighbors.
- **Avoid overlap.** If two Level-1 items have similar descriptions, the agent can't decide between them. Curate ruthlessly — bloated indexes are the failure mode.
- **Mutually-exclusive paths.** If two pieces of Level-2 detail are mutually exclusive (you'd never need both at once), split them into separate files. Don't bundle them into one SKILL.md.
- **Use the filesystem.** File trees are the simplest progressive-disclosure structure — folder hierarchies, file names, sizes, and modification times all carry signal "for free." Anthropic: *"Models are great at navigating filesystems."*
- **Let code speak.** Sometimes Level 3 is better as a runnable script than as documentation. The agent doesn't need to read it; it just runs it.

## Variants & related patterns

- [**Just-in-time context**](just-in-time-context.md) — the underlying retrieval model; PD is JIT in tiers.
- [**Structured note-taking**](structured-note-taking.md) — the agent's own outputs can be progressively disclosed (TODO.md = Level 1, notes/*.md = Level 2+).
- [**Agent Skills (SKILL.md)**](#) — the canonical productization of PD; covered in [`../skills/`](#).
- [**Code Mode (MCP tools as code)**](#) — PD applied to tool definitions; covered in [`../skills/`](#).
- [**Compaction**](compaction.md) — the compacted summary is itself a Level-1 view of prior history; notes are Level 2+.
- **MCP `search_tools` tool** — a Level-1 retrieval mechanism for very large MCP toolboxes; *"a `search_tools` tool can be added to the server to find relevant definitions"* (Anthropic).
- **Nielsen Norman's UX "progressive disclosure"** — the human-interface origin term; same principle, different audience.

## When NOT to use

- **When all data is always relevant.** A small, fixed knowledge base that fits comfortably in context — just pre-load it. PD adds tool-call overhead.
- **When tool calls are very expensive.** If each Level-2 read costs $0.10 and a turn might trigger 5 of them, you're paying $0.50 in retrieval overhead per turn. For high-volume / low-margin agents, pre-load.
- **When the model is weak at decision-making.** A model that picks the wrong Level-2 item will read the wrong content, decide wrong, and the error compounds. Use stronger models or simpler structures.
- **For sub-second latency requirements** — each level adds a round-trip. Pre-load for chatbot-style fast responses.
- **When metadata can't be made informative.** PD relies on Level-1 names + descriptions being good enough to guide selection. If your content can't be summarized in a line, PD won't work for it.
- **When you need exhaustive coverage.** If the agent *must* consider every item (audit, compliance, completeness checks), pre-loading is safer than relying on the agent to pick.

## Implementations

| Tool / framework | PD support |
|---|---|
| **Agent Skills (SKILL.md format)** | The canonical 3-level PD: metadata → SKILL.md → bundled files. Open standard since Dec 2025. |
| **Code execution with MCP** | PD via filesystem tool tree. Native pattern, no special framework needed. |
| **Claude Code** | PD via `glob`/`grep`/`read_file` + CLAUDE.md anchors. |
| **MCP `search_tools` pattern** | Build a tool that returns matching tool definitions instead of loading them all upfront. |
| **LangChain Hub / tool registries** | PD-style indexes of tools; agent fetches when relevant. |
| **LlamaIndex `RouterQueryEngine`** | A Level-1 router that dispatches to specialized retrievers (Level 2+). |
| **OpenAI Assistants `files` + `file_search`** | PD-style: files are present but content only retrieved when search hits. |
| **AutoGen `register_for_llm` with rich descriptions** | The descriptions become Level-1 metadata. |

## Companies using progressive disclosure

- **Anthropic** ✅ — explicit design principle in Skills, Code Mode, and Claude Code. ([source](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills))
- **All Agent Skills adopters** ✅ — Claude.ai, Claude Code, OpenCode, Cursor, Amp, Letta, goose, GitHub, VS Code, OpenAI Codex (Skills uses PD by design)
- **Cloudflare** ✅ — independently arrived at the same pattern via "Code Mode" (their term).
- **Cursor, Aider, Cline** ⚠ — file-tree-as-index pattern is universal in code agents.
- **Perplexity, Claude Web Search** ⚠ — search results are themselves a Level-1 view; full pages are Level 2 retrieval.

## Further reading

- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic Sept 2025
- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — Anthropic Oct 2025 (the canonical PD productization)
- [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — Anthropic Nov 2025 (PD for tools)
- [Agent Skills open standard](https://agentskills.io/specification) — Dec 2025
- [Nielsen Norman Group — Progressive Disclosure](https://www.nngroup.com/articles/progressive-disclosure/) — the UX-design origin

---

*Diagram source: [`../diagrams/src/progressive-disclosure.d2`](../diagrams/src/progressive-disclosure.d2)*
