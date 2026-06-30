# AGENTS.md

**Aliases:** project-level agent instructions, robots.txt for agents, agent README, CLAUDE.md (Anthropic variant), `.cursorrules` (Cursor variant)
**Category:** Skills & Packaging
**Sources:**
[agents.md spec](https://agents.md/) (official site; "used by over 60,000 open-source projects") ·
[OpenAI Codex CLI announcement (2025)](https://openai.com/blog/openai-codex-cli/) ·
[Cursor rules documentation](https://docs.cursor.com/context/rules-for-ai) ·
[Anthropic CLAUDE.md conventions](https://docs.claude.com/) (Claude Code documentation) ·
[Aider conventions documentation](https://aider.chat/docs/usage/conventions.html)

---

## Problem

> [!TIP]
> **ELI5.** Every coding agent (Claude Code, Cursor, Continue, Aider, Codex CLI, Cline, Replit Agent, Devin, etc.) needs to know things about your project: how to build it, how to run tests, what style to use, what NOT to touch. In 2023-2024, each tool wanted its own config file. A repo using three coding tools needed three config files — `CLAUDE.md`, `.cursorrules`, `CONVENTIONS.md` — all saying largely the same thing. **AGENTS.md** is the community standard that emerged in 2025: ONE file in the project root, free-form markdown, that any compliant coding agent reads at session start. The site agents.md says it's now used by 60,000+ open-source projects. Adopted by Cursor, Continue, Aider, Codex CLI, Cline, Roo Code, Devin, and others. Claude Code reads it too (with fallback to CLAUDE.md). Think "robots.txt for agents" — a single, predictable, agent-readable location.

The pattern is a clean example of *de-facto standardization through coordination problem solving*. No single vendor mandated it; each tool's users hated having multiple config files; the community settled on one convention, the agents.md site formalized it, and major vendors adopted it through 2025.

The naming is intentional: `AGENTS.md` is a play on `README.md` — "README for agents." The capitalization (all-caps) mimics `README` and `LICENSE`, signaling "important, conventional, top-of-repo file."

Why a single file beats per-tool configs:
- **One source of truth.** Repo conventions are stated once.
- **Tool-agnostic.** Adding a new agent doesn't require re-writing conventions.
- **PR-reviewable.** Conventions change just like code.
- **Discoverable.** New contributors see the file alongside README.
- **Future-proof.** New coding tools can adopt the format without coordination.

## How it works

> [!TIP]
> **ELI5.** Drop a markdown file named `AGENTS.md` at the project root (and optionally in subdirectories for scoped overrides). Inside, write whatever a smart new contributor would need to know to start working on the codebase — build commands, code style, architecture, gotchas, "don't touch" areas. Any agent that supports AGENTS.md reads it at session start and includes it in its system context. The closest file wins for files in subdirectories.

![AGENTS.md](../diagrams/svg/agents-md.svg)

### Location and lookup

- **Primary**: `AGENTS.md` at project root.
- **Scoped**: `AGENTS.md` in a subdirectory applies to files in that subtree.
- **Lookup**: agents walk up from the working file to find the closest `AGENTS.md`; combine with root-level.
- **Fallback chain**: many agents fall back to `CLAUDE.md`, then `.cursorrules`, then `CONVENTIONS.md` if `AGENTS.md` is absent.

This mirrors `.gitignore` / `.editorconfig` lookup semantics: predictable, hierarchical, override-friendly.

### Content (free-form markdown)

There's no required schema — it's plain markdown by design. Community conventions for what to include:

**Project overview**
- What this codebase does in 1-3 sentences
- Primary language(s) / framework(s) / runtime
- Repository layout (top-level dirs and their purpose)

**Build & run**
- How to install dependencies
- How to run tests
- How to build / deploy
- How to start dev server

**Code conventions**
- Style guide (link to formatter config)
- Naming conventions
- File organization rules
- Comment / docstring conventions

**Architecture notes**
- Key abstractions and where they live
- Cross-cutting concerns (logging, error handling, auth)
- External dependencies / services

**Agent-specific guidance**
- "Always run tests before committing"
- "Don't modify generated files in `dist/`"
- "Prefer composition over inheritance"
- "When adding endpoints, update OpenAPI spec"

**Don't-touch zones**
- Generated code paths
- Vendored dependencies
- Migration files
- Secrets / config files

**Common pitfalls**
- "Tests sometimes fail flakily on Mac — re-run twice"
- "The `_old` directory is being deprecated; don't add to it"
- "API client gen is run on commit; don't manually edit `client.ts`"

### Adoption (as of late 2025 / early 2026)

The 60,000+ project count from agents.md represents the official "adopted" status. Per-tool support:

| Tool | AGENTS.md support |
|---|---|
| **Cursor** | ✅ Native |
| **Continue** | ✅ Native |
| **Aider** | ✅ Native (plus older `CONVENTIONS.md`) |
| **Cline** | ✅ Native |
| **Roo Code** | ✅ Native |
| **OpenAI Codex CLI** | ✅ Native (announced 2025) |
| **Codex on web** | ✅ Native |
| **Claude Code** | ✅ Reads it; also reads `CLAUDE.md` |
| **GitHub Copilot Workspace** | ⚠ Adoption in progress (uses repo context) |
| **Devin (Cognition)** | ✅ Reads it |
| **Replit Agent** | ⚠ Adopting |
| **Codeium** | ⚠ Adopting |
| **Tabnine** | ⚠ Adopting |
| **Sweep** | ✅ Native |
| **OpenDevin / Open Hands** | ✅ Native |

The adoption curve in 2025 was steep — by mid-2025 AGENTS.md was the de-facto default in new coding-agent products and the migration path for older `.cursorrules` / `CLAUDE.md` repos.

### Comparison with SKILL.md

AGENTS.md and [SKILL.md](skill-md-format.md) are complementary:

| | **AGENTS.md** | **SKILL.md** |
|---|---|---|
| **Scope** | Project / repo | One task / capability |
| **Loading** | Always loaded for the project | On-demand by description match |
| **Count per repo** | 1-N (nested) | 0-many |
| **Authoritative spec** | agents.md | Anthropic Skills docs |
| **Audience** | All coding agents | Skill-aware harnesses |
| **Analogy** | README | npm package |

A typical mature repo has:
- One `AGENTS.md` at root (conventions, build, architecture)
- Possibly nested `AGENTS.md` in subdirs (scope-specific overrides)
- A `skills/` directory with per-task `SKILL.md` files (specialized capabilities)

### Writing a good AGENTS.md

Empirically, the best AGENTS.md files:

- **Start with what's surprising.** Lead with non-obvious conventions; standard practices (use Prettier; tests in `__tests__/`) can come later.
- **Are command-oriented.** "Run `npm test`" beats "tests can be run." Agents convert prose to actions.
- **Include negative examples.** "Don't add to `legacy/`" prevents costly mistakes.
- **Link to authoritative configs.** Don't restate `.prettierrc` content; link to it.
- **Stay updated.** Stale AGENTS.md causes worse outcomes than missing one.
- **Stay short.** 200-500 lines max; agents load it every session.
- **Use checkable assertions.** "Tests must pass before commit" with `npm test` command is testable.

### Anti-patterns

- **AGENTS.md as a dumping ground.** Pages of vague preferences confuse the agent.
- **Stale AGENTS.md.** Wrong info actively hurts (worse than nothing).
- **Conflicting AGENTS.md across nested dirs.** Override rules unclear; agents pick weirdly.
- **AGENTS.md with secrets.** Public repos leak; private repos still risky.
- **Over-prescriptive style rules** that contradict the formatter config.
- **No "what this repo does" section.** Agent guesses; quality varies.
- **Pure prose, no commands.** Agents need actionable steps.

### Engineering details

- **Token cost.** AGENTS.md is loaded on every session — long files cost tokens repeatedly. Use [prompt caching](../mem/prompt-caching.md) where possible.
- **Version control.** AGENTS.md lives in the repo, versioned with code. Breaking changes are PR-reviewable.
- **Team alignment.** Treat AGENTS.md updates like code reviews — convention drift is real.
- **Auto-generation possible.** Some teams generate AGENTS.md from architecture decisions (ADRs) + build scripts.
- **CI validation.** Lint AGENTS.md (link rot, command existence) in CI.
- **Multi-tool consistency.** If you also have `CLAUDE.md` or `.cursorrules`, keep them in sync or remove.

### Composition with other patterns

- **AGENTS.md + [SKILL.md](skill-md-format.md)**: standing instructions + on-demand capabilities.
- **AGENTS.md + [coding agents](../agt/coding-agents.md)**: AGENTS.md is the standing context every coding-agent session loads.
- **AGENTS.md + [maker-checker](../agt/maker-checker.md)**: AGENTS.md often defines the verification criteria.
- **AGENTS.md + [eval-driven development](../qua/eval-driven-development.md)**: include eval commands in AGENTS.md.
- **AGENTS.md + [MCP](#)** (`proto/mcp`): AGENTS.md may declare MCP servers the agent should connect to.

### Trajectory through 2026

What's emerging in late 2025 / 2026:
- **Formal schema, optional.** A YAML frontmatter section for machine-readable fields (build commands, test commands).
- **AGENTS.md in package metadata.** npm / pip / cargo could surface it for library users.
- **Per-tenant AGENTS.md.** SaaS products embed agent-friendly READMEs of their APIs.
- **AGENTS.md generators.** Tools that scan a repo and propose AGENTS.md drafts.
- **AGENTS.md linters.** Standard linting (link validity, command runnability).
- **AGENTS.md for non-coding agents.** The same pattern for research agents, data agents, etc. (already starting).

## Variants & related patterns

- [**SKILL.md format**](skill-md-format.md) — per-task capability; AGENTS.md is per-project.
- [**Code Mode**](code-mode.md) — code-mode agents read AGENTS.md for context.
- [**Coding agents**](../agt/coding-agents.md) — primary AGENTS.md consumers.
- [**Augmented LLM**](../fnd/augmented-llm.md) — AGENTS.md is one form of context augmentation.
- [**Memory architectures**](../mem/memory-architectures.md) — AGENTS.md is project-level procedural memory.
- [**Prompt caching**](../mem/prompt-caching.md) — AGENTS.md content benefits from caching.
- **CLAUDE.md** — Anthropic-specific predecessor (still works in Claude Code).
- **`.cursorrules` / `.cursor/rules/*.mdc`** — Cursor-specific.
- **`CONVENTIONS.md`** — Aider-specific.
- **`copilot-instructions.md`** — GitHub Copilot.

## When NOT to use

- **Non-code repos** where there's no agent doing code work.
- **Tiny one-file scripts** where conventions are obvious.
- **Public repos with sensitive deploy info** that shouldn't be in plaintext.
- **Highly dynamic projects** where AGENTS.md would be stale weekly.

## Implementations

| Tool | AGENTS.md handling |
|---|---|
| **Cursor** | Loads as standing context per session |
| **Continue** | Loads as system context |
| **Aider** | Loads; merges with `CONVENTIONS.md` if present |
| **Claude Code** | Loads; merges with `CLAUDE.md` if present |
| **OpenAI Codex CLI** | Loads |
| **Cline / Roo Code** | Loads |
| **Devin** | Loads |
| **Custom agents** | Trivial: read file at session start |
| **Validators / linters** | Community tools emerging |

## Companies / products using AGENTS.md

- **The 60,000+ open-source projects** ✅ (per [agents.md](https://agents.md/)).
- **OpenAI** ✅ — Codex CLI ships with AGENTS.md support.
- **Cursor** ✅ — supports AGENTS.md alongside Cursor-specific rules.
- **Continue, Aider, Cline** ✅ — adopted as primary config.
- **Anthropic Claude Code** ✅ — supports AGENTS.md and CLAUDE.md.
- **Cognition Devin** ✅ — reads AGENTS.md.
- **Sweep, OpenDevin, Open Hands** ✅ — native support.
- **GitHub** ⚠ — adoption growing in Copilot Workspace.
- **Replit, Codeium, Tabnine** ⚠ — adoption in progress.

## Further reading

- [agents.md](https://agents.md/) — canonical spec site
- [agents.md GitHub](https://github.com/openai/agents.md) — spec repo and examples (via OpenAI's mirror)
- [OpenAI Codex CLI announcement](https://openai.com/blog/openai-codex-cli/) — Codex AGENTS.md support
- [Cursor Rules for AI](https://docs.cursor.com/context/rules-for-ai) — comparable Cursor docs
- [Aider conventions](https://aider.chat/docs/usage/conventions.html) — Aider config and history
- [Anthropic Claude Code docs](https://docs.claude.com/) — CLAUDE.md and AGENTS.md support
- [The case for AGENTS.md](https://www.tigrisdata.com/blog/agents-md/) — community write-up

---

*Diagram source: [`../diagrams/src/agents-md.d2`](../diagrams/src/agents-md.d2)*
