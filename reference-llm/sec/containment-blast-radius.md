# Containment & Blast Radius

**Aliases:** sandboxing, least-privilege agents, agent isolation, blast-radius reduction
**Category:** Security & Containment
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[Anthropic — Claude Code best practices (Apr 2025)](https://www.anthropic.com/engineering/claude-code-best-practices) ·
[Simon Willison — agent security writing 2024-2026](https://simonwillison.net/) ·
[OWASP LLM Top 10 (2024-2025)](https://owasp.org/www-project-top-10-for-large-language-model-applications/) ·
[AWS — Securing generative AI applications](https://aws.amazon.com/builders-library/)

---

## Problem

> [!TIP]
> **ELI5.** Your agent is going to be wrong sometimes. Not because it's badly designed — but because it's a non-deterministic system reading untrusted inputs. The question isn't "how do I make sure it never does anything wrong?" — that's impossible. The question is: **when it goes wrong, how much damage can it do?** Containment is about answering that question deliberately. You build *concentric rings* of access — the agent runs inside the smallest ring that lets it do its job. If it goes off the rails, it can only damage what's inside that ring. That damage limit is the *blast radius*. Smaller blast radius = safer agent. Always pick the smallest ring.

By 2026, the dominant security model for agents is **not** "make the agent never make mistakes." That's intractable — LLMs are probabilistic, they read untrusted content from the web, and prompt injection attacks (see [lethal-trifecta](lethal-trifecta.md)) provably exist. The security model that works is **containment**: assume the agent *will* sometimes do the wrong thing, and bound the damage it can do *when* that happens.

This reframing matters because the alternative — relying on agent honesty, prompt engineering, or model refusals as the *primary* defense — has failed repeatedly. Every red-team exercise that probes recent agents (GPT-5, Claude Opus 4.5, Gemini 2.5 with computer use) finds at least one way to make them misbehave. Defense in depth is the operational reality.

Anthropic's [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) (April 2025) lays this out: think about *what the agent can touch* before thinking about *whether the agent will behave*. OpenAI's Operator launch (January 2025), AWS's generative-AI security guidance, and Simon Willison's running commentary all converge on the same advice.

## How it works

> [!TIP]
> **ELI5.** Draw concentric rings of access. The innermost ring has *low* trust-required: sandbox filesystem, no network, fake credentials. The outer ring has *high* trust required: real production access, real money, real customer data. Run the agent in the **smallest ring** that still lets it do its job. If a task needs only inner-ring access, never grant outer-ring permissions for it.

![Blast-radius rings](../diagrams/svg/containment-blast-radius.svg)

### The blast-radius concept

When an agent does something it shouldn't, the *blast radius* is what it can affect. The same agent in different containment configurations has dramatically different blast radii:

- **Configuration A**: Claude Code running in a developer's repo with shell access, network egress, and the developer's credentials. Blast radius: that developer's machine, their git repo (including force-push capability), any service their credentials authenticate to, the internet.
- **Configuration B**: Same Claude Code, but in a Docker container with a scoped GitHub token (read-only or single-repo write), no shell history persistence, no host filesystem mounts, and egress allowlisted to `api.github.com` + `pypi.org`. Blast radius: that container, that one repo (within token scope), nothing else.
- **Configuration C**: Same Claude Code, but in a disposable ephemeral VM with no credentials at all, working against a fork of the repo. Blast radius: that VM (destroyed after the run), zero persistent data.

Same agent, same model, same prompts. Three orders of magnitude difference in worst-case impact. The configuration matters more than the model.

### The rings, concretely

**Innermost ring — sandbox/disposable.** Highest trust granted because *nothing valuable is reachable*. Use when:
- Running untrusted code or processing untrusted input.
- Experimental / exploratory agent runs.
- Learning / fine-tuning.

Typical setup:
- Ephemeral Docker container / Firecracker microVM / Nix sandbox.
- No host filesystem mounts, or only specific read-only mounts.
- No network egress (or test-net only).
- Fake credentials / no credentials.
- Resource quotas (CPU, memory, time).

**Middle ring — scoped production.** Limited blast radius via least-privilege creds + allowlisted external comms. Use when:
- Production work, but on bounded scope.
- Agent needs *some* real-world data but not all of it.

Typical setup:
- Real credentials but scoped (per-repo GitHub token, read-only DB user, single-S3-prefix IAM role).
- Egress allowlist (specific domains/IPs only).
- Read-only where possible.
- Audit logs of every action.
- Rate limits / quotas.

**Outer ring — broad production access.** Real impact possible. Use when:
- The task genuinely requires it.
- Always with human-in-loop gating ([maker-checker](../agt/maker-checker.md)).

Typical setup:
- Full credentials (deploy keys, admin tokens, etc.).
- Network freely accessible.
- Money-moving capabilities, customer-data modifying capabilities.
- **Require explicit human approval** for high-risk actions.
- Detailed audit trail with attribution.

### The principle: smallest ring that works

The agent goes in the *smallest* ring where it can still complete the task. Not "the largest ring we'd be okay with" — the *smallest*. Concrete questions:

- Does the agent need to *write* to anything, or only read? → Read-only credentials.
- Does it need *real* data, or could it work on a copy? → Disposable copy.
- Does it need *general* internet access, or specific APIs? → Allowlist.
- Does it need credentials for *all* repos, or one? → Scoped token.
- Does it need to *run* the code, or only edit it? → Sandbox execution.

Each "yes" to "only need less" demotes the agent inward by one ring.

### Why this is the dominant 2026 security model

Three converging reasons:

1. **Prompt injection is unsolved.** LLMs cannot reliably distinguish "instructions from the user" from "instructions in an untrusted document the user asked the agent to read." [Lethal trifecta](lethal-trifecta.md) attacks remain effective against frontier models. Containment is the only known robust defense.

2. **Capability is racing ahead of alignment.** Agents in 2026 can use computers, write and execute code, browse, and chain tool calls for hours. The *cost of going wrong* keeps rising. Containment is what makes increasing capability deployable.

3. **Auditability and compliance.** Regulated industries (finance, healthcare, government) increasingly *require* documented blast-radius limits. Containment makes this provable; "the model promised to behave" does not.

### Concrete techniques

| Technique | What it bounds |
|---|---|
| **Sandboxing** (container, microVM, Nix) | Filesystem + process |
| **Egress allowlisting** ([egress-allowlisting](egress-allowlisting.md)) | Network destinations |
| **Scoped credentials** (per-repo tokens, scoped IAM) | What the agent can authenticate as |
| **Read-only mounts / DB users** | Modify capability |
| **Resource quotas** (CPU, memory, wall-clock, tokens) | Runaway cost / DOS |
| **Audit logging** | After-the-fact attribution and rollback |
| **Maker-checker / human approval** | High-impact actions |
| **Browser-as-sandbox** ([browser-as-sandbox](browser-as-sandbox.md)) | Cross-app isolation |
| **Auto-mode permission boundaries** ([auto-mode-permission-boundaries](auto-mode-permission-boundaries.md)) | Action class gating |
| **Time-bounded credentials** (STS, JIT tokens) | Temporal blast radius |

The mature setup combines several of these. Each is a separate independent boundary; defense-in-depth means several have to fail simultaneously for a breach to do real damage.

### Common anti-patterns

- **Giving the agent your full credentials** for convenience. Anti-pattern. Use scoped tokens.
- **Running agents directly on your dev machine** with shell access. Common (developer convenience), but treat as outer ring.
- **No egress restrictions.** A breached agent with full network egress can exfiltrate anything it can read.
- **Mounting your home directory** into the agent container. Defeats the point of the container.
- **Reusing the same long-lived API key across many agent runs.** No way to attribute or revoke.
- **Trusting the model to "decline" risky actions.** Defense-in-depth means the model's refusal is one layer among several.
- **Auto-mode (no human approval) for outer-ring actions.** Either move the action inward or gate it with approval.

### Containment in practice: examples

**Claude Code in a developer environment.** Anthropic recommends running in a container or sandbox; explicit "I'd prefer to NOT run this on your laptop" guidance in their docs. Permission prompts before tool execution.

**OpenAI Codex (cloud)** runs in cloud sandboxes by design — each session is an ephemeral environment.

**Devin (Cognition)** runs in isolated cloud sandboxes; doesn't have access to the customer's machine.

**ChatGPT Operator** runs in a remote browser sandbox; the user's local browser isn't compromised even on a misbehaving agent run.

**Cursor's "Agent" mode** prompts the user before destructive shell commands; recent versions have stricter defaults.

**Computer-use agents** ([../agt/computer-use.md](../agt/computer-use.md)) almost universally run in disposable VMs.

### The economics

A useful framing: containment is **insurance** that scales with capability. Cheap when the agent's capabilities are low (you don't need much defense for a chatbot that just answers questions). Expensive when the agent has broad real-world capability (you must invest in sandboxing, scoped creds, audit). Pay the cost proportional to the risk.

The cost of getting this wrong scales superlinearly with capability — a misbehaving 2023 chatbot embarrasses you; a misbehaving 2026 computer-use agent with admin credentials can do material harm.

## Variants & related patterns

- [**Lethal trifecta**](lethal-trifecta.md) — the threat model containment defends against.
- [**Egress allowlisting**](egress-allowlisting.md) — network-level ring boundary.
- [**Browser-as-sandbox**](browser-as-sandbox.md) — using the browser security model itself.
- [**Auto-mode permission boundaries**](auto-mode-permission-boundaries.md) — graduated trust at runtime.
- [**Maker-checker**](../agt/maker-checker.md) — human-in-loop for outer-ring actions.
- [**Computer use**](../agt/computer-use.md) — sandbox-by-default for safety reasons.
- **OWASP LLM Top 10** — the broader threat catalog containment partly addresses.
- **Capability-based security** (POLA, OS-level analog) — informs the agent ring model.

## When NOT to use (or relax)

- **Local dev exploration** — full sandbox is overhead; developers often work in middle ring with awareness.
- **Read-only research agents** — if the agent can't *write* anywhere, blast radius is naturally limited.
- **Single-purpose narrow assistants** without tool use — the threat model collapses to standard LLM safety.

But: as soon as the agent can *act* (tools, code execution, network calls), containment becomes essential.

## Implementations

| Tool / platform | Containment story |
|---|---|
| **Docker / Podman** | Filesystem + process sandbox, easy to set up |
| **Firecracker / Kata Containers** | Hardware-isolated microVMs |
| **gVisor** | Sandboxed kernel; good middle ground |
| **Nix flake sandboxes** | Build-system-level isolation |
| **E2B, Modal, Daytona** | Hosted sandboxes-as-a-service for agents |
| **GitHub Codespaces** | Disposable dev environments |
| **AWS Bedrock Agents + STS** | Time-bound credentials |
| **GCP Workload Identity** | Scoped identity for agents |
| **Anthropic Workbench** | Encourages sandboxed deployment |
| **OpenAI Codex (cloud sandboxes)** | Built-in containment |

## Companies / projects exemplifying containment-first design

- **Anthropic** ✅ — Claude Code docs explicitly recommend sandboxed execution.
- **OpenAI** ✅ — Codex (cloud), Operator (remote browser), both sandbox-by-default.
- **Cognition (Devin)** ✅ — runs in isolated cloud VMs.
- **Cursor, Replit, Aider** ⚠ — increasingly add permission gating and sandbox modes.
- **E2B, Modal, Daytona** ✅ — productize sandboxes-for-agents.
- **AWS, GCP, Azure** ✅ — Bedrock / Vertex / Foundry agent infrastructure with scoped IAM.
- **Anthropic's MCP host implementations** ✅ — scope tool access per server.
- **Various Fortune-500 internal deployments** ⚠ — regulated industries operating agents inside strict containment.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024
- [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic Apr 2025
- [Lethal Trifecta (Simon Willison)](https://simonwillison.net/tags/lethal-trifecta/) — running commentary on prompt-injection threats
- [OWASP Top 10 for LLM Applications (2024-2025)](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [AWS Builders' Library — generative AI security](https://aws.amazon.com/builders-library/)
- [Saltzer & Schroeder — The Protection of Information in Computer Systems (1975)](https://www.cs.virginia.edu/~evans/cs551/saltzer/) — original "least privilege" formulation
- [Computer-use sandbox best practices](https://www.anthropic.com/news/3-5-models-and-computer-use) — Anthropic Oct 2024

---

*Diagram source: [`../diagrams/src/containment-blast-radius.d2`](../diagrams/src/containment-blast-radius.d2)*
