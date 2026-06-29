# Auto-Mode Permission Boundaries

**Aliases:** graduated permissions, action-tier gating, YOLO mode boundaries, agent action classification, allow/notify/gate/approve tiers
**Category:** Security & Containment
**Sources:**
[Anthropic — Claude Code best practices (Apr 2025)](https://www.anthropic.com/engineering/claude-code-best-practices) ·
[Cursor documentation on agent permissions (2024-2026)](https://cursor.sh/) ·
[Cline / Aider permission models (OSS, 2024-2026)](https://github.com/cline/cline) ·
[GitHub Copilot Workspace docs](https://github.blog/news-insights/product-news/github-copilot-workspace/) ·
production practice across coding agents and computer-use agents

---

## Problem

> [!TIP]
> **ELI5.** Not all agent actions are equally risky. Reading a file is fine. Deleting your database is not. If you ask the user to approve every single action, you've made the agent useless (death by 1000 prompts). If you skip permission checks entirely, one bad decision wrecks things. The fix: **classify actions into tiers — auto-allow safe ones, notify on medium-risk, require approval for dangerous, never auto for irreversible.** This is "graduated permissions" — the dominant UX pattern in 2026 coding agents (Cursor, Claude Code, Aider, Cline) and computer-use agents (Operator). Get the tiers right and the agent feels fast *and* safe; get them wrong and it feels either annoying or terrifying.

The early agent UX (2023-2024) had two failure modes:

- **"Confirm every action" mode**: every tool call requires user approval. Safe, but painfully slow. Users disable the prompts within an hour. Effectively useless for autonomous work.
- **"Full auto" mode (YOLO mode)**: agent acts freely without permission. Fast and powerful, but high blast radius. One mistake — a `rm -rf`, a force-push, a misdirected email — is catastrophic.

By 2025-2026, mature agent products converged on **graduated permission tiers**. Not all actions are treated the same. Reversible, low-impact actions auto-execute. Irreversible or high-impact actions require explicit human approval. Medium-risk actions are notified but proceed. This UX makes the agent fast for safe work and safe for dangerous work — usable in production.

Anthropic's [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) describes this explicitly. Cursor, Aider, Cline, Claude Code's own permission system, Copilot Workspace, Codex CLI — all converge on roughly the same tier structure. By 2026 it's an emergent standard, not yet formalized but consistent across products.

## How it works

> [!TIP]
> **ELI5.** Four tiers: **(1) Auto-allow** for reversible, low-impact stuff (read files, search). **(2) Notify + auto** for things you should know about but the agent can keep going (write files in workspace). **(3) Gate** with a prompt for stuff that's hard to undo (git push, shell commands). **(4) Require explicit approval** for irreversible or high-stakes actions (deploy prod, send money, drop tables). Configure once per project; the agent respects it forever.

![Auto-mode permission tiers](../diagrams/svg/auto-mode-permission-boundaries.svg)

### The four tiers, with examples

**Tier 1: Auto-allow.** No prompt, no notification. Agent proceeds silently. Includes:
- Read files in the workspace.
- Run linters / formatters.
- Search the web (read-only).
- Query allowlisted APIs.
- Open IDE panels.
- Compute / search operations with no side effects.

Property: **reversible, low impact, no data leakage risk**. If wrong, no damage.

**Tier 2: Notify + auto.** Agent does the thing, but user is informed (toast, log line). Includes:
- Write or edit files in the workspace.
- Install dev dependencies.
- Run tests.
- Modify ignored / build files.
- Local-only side effects in a sandbox.

Property: **reversible (file undo, git reset), limited to workspace, agent should keep moving**. User notified for awareness, not blocking.

**Tier 3: Gate.** Prompt before action. Agent waits for explicit "yes". Includes:
- `git commit` / `git push`.
- Shell commands with elevated privileges (sudo, system-level changes).
- Modifying config files or environment vars.
- Cross-system writes (writing to a database, modifying another service).
- Cross-repo operations.
- Sending non-critical external comms.

Property: **hard to reverse, or visible to other parties (PRs, commits, deployments)**. Worth a second of human friction to prevent surprise.

**Tier 4: Require explicit approval.** Strong gating; cannot be set to auto-approve without explicit per-action confirmation. Includes:
- Deploy to production.
- Money-moving operations.
- Send external email or post publicly.
- Delete data, drop database tables.
- Change IAM permissions or org membership.
- `git push --force`, rewriting history.
- Send sensitive data outside the company.

Property: **irreversible OR high external consequences**. Must require human gatekeeping; auto-approval modes don't apply.

### Configuration model

The dominant 2026 pattern:

1. **Defaults shipped by the tool.** Cursor, Claude Code, etc. ship with sensible defaults for each tier.
2. **Per-project overrides.** Stored in repo config (`.cursor/permissions.json`, `.claude/permissions.md`, `.cline/permissions.yaml`, etc). Allow project teams to set their own rules.
3. **Per-user overrides.** A specific developer can be more or less permissive.
4. **Learning over time.** "Always allow this" / "Always deny this" prompts persist user choices.
5. **Temporary YOLO mode.** Some tools allow temporary trust elevation ("for the next 10 minutes, auto-approve tier 3") — useful for trusted contexts (e.g., known-safe ephemeral sandbox).
6. **Auditable.** All actions logged, including auto-allowed ones, for post-hoc review.

### What goes in which tier — design principles

The classification isn't arbitrary. Principles that drive it:

- **Reversibility.** Can you undo this in seconds, minutes, hours, or never? → Tier rises with reversibility cost.
- **Externality.** Does this affect only you / your sandbox, or does it leave a trace visible to others (PRs, commits, emails, deployments)? → Higher external visibility = higher tier.
- **Data exposure.** Could this leak private data? → Read of public data: tier 1. Read of private data: tier 2-3 depending on destination. Send private data externally: tier 4.
- **Cost / resource impact.** Free or trivial: tier 1. Significant resources (large LLM call, paid API): tier 2-3.
- **Authority assumed.** Local action: lower tier. Action under user's external identity (commit-as-user, send-email-as-user): higher tier.

In edge cases, **default upward, not downward** — when uncertain, classify as higher tier. The cost of an extra approval is a few seconds; the cost of an unapproved disaster is hours-days.

### Notable specific examples

**Claude Code's permission model:**
- File reads, web searches: auto.
- File writes within workspace: auto with logging.
- Shell commands: gated by default, can be configured with allow-lists.
- Sensitive shell operations (`git push --force`, `rm -rf`): require explicit prompt.
- Auto-approve toggles available but defaults are conservative.

**Cursor's "Agent" mode:**
- Reads and small edits: auto.
- Multi-file refactors: notify with diff preview.
- Shell commands: prompt by default.
- Destructive operations: require explicit per-action confirmation.

**Aider (OSS):**
- File edits via diff: shown to user pre-commit.
- Commits: configurable auto vs prompt.
- Shell: gated by default.

**Cline (OSS):**
- Tiered system with per-tool granularity.
- "Auto-approve" toggles per-tool, persistent.
- Explicit "destructive operation" categorization.

**OpenAI Operator:**
- Most browser actions: auto.
- Per-site authorization: explicit user grant.
- Payment / sensitive actions: explicit per-action confirmation.

**Computer-use agents (Anthropic's reference):**
- Mouse/keyboard actions in sandbox: auto.
- File system in sandbox: auto.
- Outside-sandbox effects: not supported in reference.

### Common anti-patterns

- **Universal auto-approve.** Tempting for prototypes; catastrophic in production. Use per-tier configs.
- **Universal manual approval.** Users disable it within an hour. Then the agent has *no* gating because the user is rubber-stamping.
- **Cluttered prompts.** Every prompt should clearly say *what* is about to happen and *why*. Vague "Tool wants to execute, allow?" prompts get ignored.
- **No way to learn.** If the user has to re-approve the same action 50 times, they'll disable prompts. Always offer "always allow for this project" with reasonable scope.
- **Hidden tier escalations.** If the agent can secretly do a tier-3 action via a tier-1 tool, the gating is bypassed. Tool design should match tier classification.
- **No audit log.** Even auto-approved actions should be logged. When something goes wrong, the log is how you understand it.
- **Treating all "shell commands" as one tier.** `ls` vs `rm -rf` should not be the same tier. Smart tools classify within the shell tool.

### How this composes with other security controls

Permission boundaries are **runtime, application-layer** gating. They compose with:

- **[Containment / blast radius](containment-blast-radius.md)** — the agent's *capability* is bounded by sandbox + creds; permission boundaries bound the *use* of those capabilities.
- **[Egress allowlisting](egress-allowlisting.md)** — network-layer; even if a permission check is bypassed, the network won't let the data out.
- **[Maker-checker](../agt/maker-checker.md)** — applies specifically to high-stakes outputs; tier-4 actions are usually maker-checker'd.
- **[Browser-as-sandbox](browser-as-sandbox.md)** — provides the container; permission boundaries gate the actions within it.

Defense-in-depth means *several* of these are active simultaneously. Permission boundaries are the most user-visible layer, but they're not the only one.

### Communicating tier in the UI

UI design matters more than the back-end classification. Best practices that have emerged:

- **Distinct visual treatment per tier.** Calm color for tier-1 (or invisible), neutral for tier-2 notifications, warning yellow for tier-3 gates, red for tier-4 approvals.
- **Diff preview for file changes.** Show what's about to change before the action runs.
- **Command preview for shell.** Show the exact command, not a paraphrase.
- **Resource estimate.** "This will write to 8 files / send 2 emails / commit 3 changes" before user confirms.
- **Reasoning context.** Why the agent wants to do this; one-sentence explanation alongside the prompt.
- **One-click "always allow this exact command"** with clear scope (this session / this project / forever).

### Calibration over time

A team using an agent develops habits — they learn which tier classifications are appropriately conservative and which are over-prompting. The product should support this:
- Telemetry on how often tier-3 prompts are approved (if 99%, maybe move to tier-2).
- Telemetry on how often tier-2 actions cause issues (if frequent, maybe move to tier-3).
- Per-user learning so different developers calibrate independently.

The tier classification is not static — it's *a starting point* that should evolve with the team.

## Variants & related patterns

- [**Containment & blast radius**](containment-blast-radius.md) — the architectural pattern this UX implements.
- [**Lethal trifecta**](lethal-trifecta.md) — tier 4 specifically gates the exfiltration consequence.
- [**Egress allowlisting**](egress-allowlisting.md) — network-layer co-defense.
- [**Browser-as-sandbox**](browser-as-sandbox.md) — sandbox layer below; permissions layer above.
- [**Maker-checker**](../agt/maker-checker.md) — formal version of tier-4 approval.
- [**Workflows vs agents**](../agt/workflows-vs-agents.md) — workflow steps naturally map to permission boundaries.
- **Capability-based security** — the deeper computer-science principle.
- **OAuth scope granularity** — analogous pattern in API authorization.
- **POLA (Principle of Least Authority)** — the underlying security principle.

## When NOT to use (or simplify)

- **Pure research / dev exploration with no real impact** — can use blanket auto-approve in fully isolated sandbox.
- **Single-user fully-trusted contexts** — heavy UI gating may be unnecessary friction; log instead.
- **Background batch agents** — no human present to approve; tier 4 actions should be excluded entirely (move to maker-checker async pattern).

Even in these cases: *log* everything, even if you don't prompt. The audit trail matters.

## Implementations

| Tool | Permission UX |
|---|---|
| **Cursor Agent mode** | Tiered, diff previews, configurable per-project |
| **Anthropic Claude Code** | Tiered, configurable, log-everything |
| **GitHub Copilot Workspace** | Plan-then-act with diff review per step |
| **Aider** | Diff preview, commit prompts, configurable |
| **Cline** | Per-tool auto-approve toggles, destructive-op categorization |
| **OpenAI Codex CLI** | Action gating with previews |
| **OpenAI Operator** | Per-site authorization + per-action confirms for sensitive ops |
| **Anthropic computer use** | Sandbox + per-action confirms in reference impl |
| **Devin (Cognition)** | Async approval flow for sensitive ops |
| **HumanLayer** | Productizes the approval flow as a service |

## Companies / projects with mature permission-tier UX

- **Anthropic** ✅ — Claude Code permissions explicitly documented.
- **Cursor** ✅ — agent permissions configurable per-project.
- **OpenAI** ✅ — Operator's site authorization + Codex CLI tiers.
- **Cognition (Devin)** ⚠ — async approval flow for sensitive ops.
- **Aider, Cline, Continue.dev** ✅ — OSS implementations of tiered permissions.
- **GitHub Copilot** ✅ — Workspace plan-then-act with step-level approval.
- **HumanLayer** ✅ — productizes approval workflow as a service.
- **Replit Agent** ✅ — has tiered permission UX.

## Further reading

- [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic Apr 2025
- [Introducing Operator](https://openai.com/index/introducing-operator/) — OpenAI Jan 2025
- [Cline permission docs](https://github.com/cline/cline) — OSS reference
- [Aider scripting and approval modes](https://aider.chat/docs/scripting.html)
- [HumanLayer](https://www.humanlayer.dev/) — approval-flow-as-a-service
- [POLA — Principle of Least Authority](https://en.wikipedia.org/wiki/Principle_of_least_privilege) (Wikipedia)
- [Saltzer & Schroeder — Protection of Information in Computer Systems (1975)](https://www.cs.virginia.edu/~evans/cs551/saltzer/)

---

*Diagram source: [`../diagrams/src/auto-mode-permission-boundaries.d2`](../diagrams/src/auto-mode-permission-boundaries.d2)*
