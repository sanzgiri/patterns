# SKILL.md Format (Anthropic Skills)

**Aliases:** Agent Skills, Claude Skills, packaged skills, portable agent capabilities, skill folders
**Category:** Skills & Packaging
**Sources:**
[Anthropic — Introducing Agent Skills (Oct 16, 2025)](https://claude.com/blog/skills) ·
[Anthropic Skills documentation](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) ·
[Anthropic cookbook — Skills](https://github.com/anthropics/anthropic-cookbook/tree/main/skills) ·
related: pi skills, Claude Code skills (similar conventions), Cursor `.cursor/rules/`, Continue prompts

---

## Problem

> [!TIP]
> **ELI5.** An agent that's good at one task — generating PDFs, deploying to Vercel, querying a specific API — needs a specific recipe: a prompt, maybe some helper scripts, examples, and edge-case notes. Without packaging, that recipe lives in someone's head (or scattered across configs and chat history) and disappears when the team changes. **SKILL.md** is Anthropic's October 2025 standard for a *portable folder* containing everything an agent needs to do one kind of task well: a markdown file with structured frontmatter (the "skill manifest"), a body of instructions (the SOP), and optional adjacent files (scripts, templates, deeper reference). The agent loads skills on demand — it scans skill descriptions cheaply, matches the user's intent, then loads only the matched skill's body. Result: a way to *share, version, and reuse* agent capabilities across people, projects, and harnesses.

The October 16, 2025 Anthropic post [Introducing Agent Skills](https://claude.com/blog/skills) made the pattern official. The core insight: a good agent isn't a single big prompt — it's a *catalog* of specialized capabilities the agent can apply when relevant. SKILL.md is the format for each entry in the catalog.

Before SKILL.md, the same problem was solved ad-hoc:
- Claude Code: `CLAUDE.md` in the project root, plus chat-history conventions
- Cursor: `.cursorrules` (single file) → `.cursor/rules/*.mdc` (multiple files with frontmatter)
- Continue: `.continue/config.json` + prompts
- Aider: `.aider.conf.yml` + `CONVENTIONS.md`
- Each agent product: a slightly different scheme

SKILL.md proposes a *common, portable format* — adopted first by Claude Code and the Anthropic ecosystem (including pi, which uses essentially the same convention).

## How it works

> [!TIP]
> **ELI5.** A skill is a *folder*. Inside is `SKILL.md` (mandatory) with a YAML frontmatter at the top — most importantly a `description` field that tells the router *when* this skill should fire. Below the frontmatter is the prose body: the actual instructions, workflow, examples, anti-patterns. Beside `SKILL.md` you can have scripts, templates, sample inputs/outputs, reference docs. The agent loads the folder when its description matches the user's intent — progressive disclosure means descriptions are always-cheap-loaded but the rest only loads on a hit.

![SKILL.md format](../diagrams/svg/skill-md-format.svg)

### Folder structure

```
skills/
  generate-pdf/
    SKILL.md          # required: frontmatter + body
    scripts/
      render.py       # helper script
    templates/
      report.html     # template asset
    examples/
      sample.md       # example input/output
    reference.md      # deeper reference (loaded if needed)
```

The convention is "everything for this skill lives here." A user adding a new skill copies the folder; removing one deletes the folder. No central registry. No global config to edit.

### SKILL.md frontmatter

The standard frontmatter (Anthropic's October 2025 spec):

```yaml
---
name: generate-pdf
description: |
  Generate a PDF report from a markdown source file using the
  custom company template. Use when the user asks to make a
  "monthly report", "client deliverable", or "branded PDF".
---
```

Required fields:
- **`name`**: unique identifier (kebab-case typical)
- **`description`**: one paragraph (often 2-3 sentences) describing *when this skill should fire*

Optional fields (depending on harness):
- `version`, `author`, `license`
- `inputs` / `outputs` (typed I/O schemas)
- `dependencies` (other skills required)
- `tools` (allowed tool list)
- `tags` (categorization)

The `description` field is the load-bearing piece. The router compares it to the user's intent (often via embedding or LLM classifier) to decide whether to load the skill. A vague description means the skill won't fire when it should — or fires when it shouldn't. Good descriptions are specific about triggers, mentioning the exact phrasing or task category.

### SKILL.md body

Below the frontmatter, free-form markdown. By community convention this includes:

- **What the skill does** (one-paragraph summary)
- **Workflow steps** (numbered SOP)
- **Tool calls** (which tools to use, how to invoke them, with examples)
- **Templates** (file paths to templates the agent should use)
- **Edge cases** (gotchas, what to verify)
- **Anti-patterns** (what NOT to do)
- **Verification** (how to confirm success)

The body is essentially a runbook *for an LLM agent*, written in the same language a senior engineer would write for a junior — assumed competence on basics, specific on the domain particulars.

### Progressive disclosure (the load model)

Skills aren't all loaded at once. The router-load-execute pattern:

1. **Startup**: scan all skills' folders, build an index of `(name, description)` pairs. This index is small — typically 100-500 tokens per skill — and is always available to the agent.

2. **User query arrives**: the router (an LLM call or embedding lookup) compares the query to skill descriptions. Matches one or more skills.

3. **Load matched skill body**: the SKILL.md body is read into context. Now the agent knows the full instructions.

4. **Lazy-load adjacent files**: scripts, templates, reference docs load only if the body says to use them (mentioned by relative path).

This means a project with 100 skills doesn't pay for 100 skill prompts on every query — only the relevant skill(s) load. It's [just-in-time context](../ctx/just-in-time-context.md) applied to capabilities.

### Why portable matters

A skill written for Claude Code should work in pi, in a custom agent harness, in a CI bot — without re-writing. The format is plain markdown with conventional frontmatter, so any harness can parse it.

Production benefits:
- **Skills are reviewable in PRs** (just markdown changes)
- **Skills are testable** (give the agent the skill + a known task; check output)
- **Skills are versionable** (git history)
- **Skills are shareable** (publish a folder; others copy it)
- **Skills compose** (one skill calls another via "see also" references)

The 2025-2026 ecosystem now has skill *libraries* — collections of skills published on GitHub for common tasks (code review, documentation generation, deployment, data extraction). Like npm packages, but for agent capabilities.

### Anthropic Skills launch — what's new

The October 2025 Anthropic launch positioned Skills as a first-class platform feature:

- **Skills work in Claude Code, Claude Desktop, and the API.** Same format, multiple consumption surfaces.
- **Skills are discoverable** via a marketplace-style listing (Anthropic-curated + community).
- **Skills can call MCP servers**, making them composable with the broader MCP ecosystem.
- **Skills can invoke other skills**, allowing hierarchical composition.

This is the moment "skill packaging" moved from convention to platform feature.

### Comparison to alternatives

| Format | Scope | Loading | Portable |
|---|---|---|---|
| **SKILL.md** | Per-task capability | On-demand by description match | Yes (open spec) |
| **AGENTS.md** (see [agents-md](agents-md.md)) | Project-level standing instructions | Always loaded for that repo | Yes (open spec) |
| **CLAUDE.md** | Project-level (Claude Code) | Always loaded | Claude Code primarily |
| **`.cursor/rules/*.mdc`** | Per-rule | On-demand by file glob | Cursor-only |
| **`CONVENTIONS.md`** (Aider) | Project-level | Always loaded | Aider-specific |
| **Custom system prompts** | Per-app | Always loaded | App-specific |

SKILL.md is the most fine-grained: each skill is a separate folder, loaded only when relevant.

### Writing good skill descriptions

The description is the router's signal. Common failure modes:

- **Too vague.** "Helps with code." → fires on everything or nothing.
- **Too specific.** "Generates PDFs from our v2 invoice template using LaTeX." → won't fire on "make a PDF" generally.
- **Missing trigger phrases.** Routers often match on user phrasing; include common phrasings.
- **No anti-trigger.** Add "Do NOT use this for X" if there's an ambiguous adjacent case.

A good description (Anthropic-style):

```yaml
description: |
  Use this skill when the user asks to generate a PDF report
  from markdown content. Triggered by phrases like "generate
  report", "create PDF", "make a deliverable", or "export as PDF".
  Do NOT use for non-PDF formats (HTML, DOCX) - those have
  separate skills.
```

### Engineering details

- **Skill discovery**: typical mechanism is a `skills/` directory scan at agent startup. Some harnesses watch for changes.
- **Skill conflicts**: if two skills match a query, the router may load both or pick the best match. Define a tie-breaking rule.
- **Skill versioning**: include `version:` in frontmatter; major changes can rename.
- **Skill testing**: a skill repo should include a small eval set per skill — known inputs and expected outcomes.
- **Skill metrics**: track per-skill firing rate, success rate, user feedback.
- **Skill security**: skills may include scripts that execute; review just like code dependencies.
- **Skill governance**: who can add/modify? Usually the same review process as code.

### Anti-patterns

- **One mega-skill** for everything → no progressive disclosure benefit.
- **Skills that duplicate base prompt content** → wastes tokens on repeated info.
- **Mutually contradictory skills** → router picks wrong; user gets inconsistent behavior.
- **Skills with secrets** (API keys, credentials) → leaks in source control.
- **No skill descriptions** → router can't find the right skill.
- **Adjacent files referenced by absolute path** → breaks portability.
- **Markdown body with no examples** → agent guesses; quality varies.

### When skills don't fit

- **Truly one-shot agents** (single fixed task): just use the system prompt.
- **Very small skill set** (1-3 capabilities): system prompt sections suffice.
- **Highly dynamic capabilities** (changes per user) → runtime config more appropriate.
- **Capabilities requiring training** (fine-tuning) → skills are prompts, not weights.

## Variants & related patterns

- [**AGENTS.md**](agents-md.md) — project-level shared instructions; complementary.
- [**Code Mode**](code-mode.md) — skills often package code that gets executed.
- [**MCP** (`proto/mcp`)](#) — MCP servers expose tools; skills orchestrate them.
- [**Progressive disclosure**](../ctx/progressive-disclosure.md) — the loading principle.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — skills are JIT capabilities.
- [**Memory architectures**](../mem/memory-architectures.md) — skills are procedural memory.
- [**Augmented LLM**](../fnd/augmented-llm.md) — skills augment via instructions + tools.
- [**Single agent with tools**](../agt/single-agent-with-tools.md) — skills compose with tool use.
- **Cursor `.cursor/rules/`** — analogous, Cursor-specific.
- **`CLAUDE.md`** — older Claude Code convention.

## When NOT to use

- **One-prompt apps** where there's no real catalog to manage.
- **Apps without skill-aware harnesses** — SKILL.md is just markdown if nothing loads it.
- **Capabilities that need training/weights** — skills are runtime instructions.
- **Privacy-strict deployments** where skill content can't live alongside code.

## Implementations

| Tool | SKILL.md support |
|---|---|
| **Claude Code** | Native (Oct 2025) |
| **Claude Desktop** | Via Anthropic Skills (Oct 2025) |
| **Anthropic API** | Skills as first-class object |
| **pi (this harness)** | Native (`~/.pi/agent/skills/`) |
| **OpenAI Agents SDK** | Adopting similar pattern (Q4 2025+) |
| **Mastra / Vercel AI SDK** | Custom; convention emerging |
| **Custom harnesses** | Easy — markdown + frontmatter parsing |

## Companies / products using skill-md format

- **Anthropic** ✅ — created and ships the format ([Oct 2025 launch](https://claude.com/blog/skills)).
- **Claude Code users** ✅ — broad adoption among engineers using Claude in IDE.
- **pi (Earendil Works)** ✅ — uses essentially the same convention.
- **Cursor** ⚠ — adopting compatible format for `.cursor/rules/` (similar shape).
- **Continue, Aider, Cline, Roo Code** ⚠ — convergence on the format.
- **OpenAI Codex CLI** ⚠ — adopting compatible conventions in Q4 2025.
- **Community skill libraries** ⚠ — emerging on GitHub.

## Further reading

- [Introducing Agent Skills](https://claude.com/blog/skills) — Anthropic Oct 2025
- [Agent Skills documentation](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) — Anthropic docs
- [Anthropic Skills cookbook](https://github.com/anthropics/anthropic-cookbook/tree/main/skills) — examples
- [AGENTS.md spec](https://agents.md/) — complementary project-level format
- [pi skills documentation](https://github.com/earendil-works/pi) — pi harness
- [Cursor rules documentation](https://docs.cursor.com/context/rules-for-ai) — similar pattern

---

*Diagram source: [`../diagrams/src/skill-md-format.d2`](../diagrams/src/skill-md-format.d2)*
