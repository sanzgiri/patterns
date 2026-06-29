# Context Plumbing & Reinforcement

**Aliases:** context routing, source-sink architecture; reinforcement = state echoing, objective re-injection, ambient memory
**Category:** Context Engineering
**Sources:**
[Matt Webb — Context plumbing (Nov 2025)](https://interconnected.org/home/2025/11/29) ·
[Armin Ronacher — Agent design is still hard (Nov 2025)](https://lucumr.pocoo.org/2025/11/22/agent-design/) ·
[Anthropic — Claude Code TODO list pattern](https://www.anthropic.com/engineering/claude-code-best-practices) (implicit reinforcement) ·
12-Factor Agents — Factor 9 (compact errors into context)

---

## Problem

> [!TIP]
> **ELI5.** Two related problems that show up in any non-trivial agent. (1) **Context plumbing**: useful information appears all over the place — your emails, calendar, edited files, sensor readings, an MCP server somewhere — but the AI needs it in one specific spot at the right time. Someone has to do the pipework. (2) **Reinforcement**: even with the right context loaded, agents *forget the goal* mid-loop and chase whatever the most recent tool result was. You have to keep re-reminding them of the original objective at every turn.

Both of these are 2025 frames for problems the field had been hitting since 2023, now with explicit names. They sit alongside [compaction](compaction.md), [just-in-time context](just-in-time-context.md), and [structured note-taking](structured-note-taking.md) as the four practical levers of context engineering — but they operate at a different level of abstraction. Where compaction and JIT are about *what's in the window right now*, plumbing is about *how stuff gets to the window in the first place*, and reinforcement is about *keeping the agent oriented across many turns*.

## Context plumbing (Matt Webb, Nov 2025)

> [!TIP]
> **ELI5.** Think of an AI agent as a tap. Water comes out where the tap is. But the water itself comes from reservoirs far away — emails on a server, files on disk, sensor readings, API state. The pipework that gets the right water to the right tap, at the right moment, is the engineering work. Matt Webb calls it context plumbing.

In a [November 2025 post](https://interconnected.org/home/2025/11/29), Matt Webb proposed thinking about AI system architecture as **plumbing the sources and sinks of context**:

> *"Context appears at disparate sources, by user activity or changes in the user's environment: what they're working on changes, emails appear, documents are edited, it's no longer sunny outside, the available tools have been updated. This context is not always where the AI runs (and the AI runs as close as possible to the point of user intent). So the job of making an agent run really well is to move the context to where it needs to be."*

![Context plumbing — sources and sinks](../diagrams/svg/context-plumbing.svg)

The model has three components:

**Sources** — places where context originates. User actions (typed messages, edited files, clicks, voice). Environment changes (new emails, calendar updates, file edits by other apps, weather, IoT sensors). Tool/system updates (new MCP servers registered, schemas changed, RBAC permissions adjusted). External state (database rows changed, API states updated, jobs completed).

**Sinks** — places where the AI actually runs and consumes context. These are *agents*, plural — different specialists running in different places: a coding agent in your IDE, a research agent in a chat UI, a scheduling agent in a webhook handler, an email summarizer in a daemon.

**Plumbing** — the engineering layer between them. Watch or poll the sources. Filter, dedupe, format, time-window. Route the right context to the right sink at the right time. This is the actual systems-engineering work of building production agents — and Webb's contribution is to give it a name distinct from "prompt engineering" or "RAG."

### Why the framing matters

Through 2024, most agent architectures assumed context was either *user input* or *retrieved-via-RAG*. The plumbing frame makes explicit that:

- **Context has lifecycle.** It's created somewhere, may be transformed by intermediary systems, eventually lands in an LLM call, sometimes affects external state.
- **Different agents need different slices.** A coding agent doesn't need calendar events; a scheduler doesn't need your codebase. Centralized "shovel everything at one big model" architectures lose to specialized agents with curated inputs.
- **The plumbing IS the product** for many enterprise agent systems. Choosing what to surface, when, with what summarization, is half of what makes a useful agent.
- **Reactivity matters.** Some context is push (events fire and need handling); some is pull (the agent asks for it). Mixing them well is non-trivial.

In practice, context plumbing is implemented with event buses, watchers, MCP servers, scheduled summarizers, change-data-capture pipelines, webhooks, and increasingly **sub-agents whose entire job is to maintain a curated context for a sibling agent** (see [`../agt/`](#) for sub-agent patterns).

## Reinforcement (Armin Ronacher, Nov 2025)

> [!TIP]
> **ELI5.** When the agent calls a tool — say, reads a file — and you send the result back, don't *just* send the result. Also send a reminder: "Here's what we were trying to do. Here's the TODO list. Here's the constraint we discovered earlier." Otherwise the agent gets fixated on the most recent thing it saw and drifts off the goal. It's like coaching a colleague who's deep in the weeds: every few minutes you re-state the objective.

Armin Ronacher introduced the term [reinforcement](https://lucumr.pocoo.org/2025/11/22/agent-design/) in a November 2025 post drawing lessons from building production agents:

> *"Every time the agent runs a tool you have the opportunity to not just return data that the tool produces, but also to feed more information back into the loop. For instance, you can remind the agent about the overall objective and the status of individual tasks. Another use of reinforcement is to inform the system about state changes that happened in the background."*

![Reinforcement — enriched tool returns](../diagrams/svg/reinforcement.svg)

The mechanic: instead of returning the raw output of a tool call, wrap it with **ambient context** the agent needs to make its next decision well. Typical reinforcement payloads:

- **The original objective** (1-2 sentences, restated)
- **Current TODO state** (what's done, what's pending, what's blocked)
- **Recently-discovered constraints** ("note: this codebase requires Python 3.10")
- **Background state changes** ("file X was modified by another process while you were working")
- **Token / time budget remaining**
- **Permissions or scope reminders** ("you are not authorized to modify files in /prod")

Why it works: an LLM in a long loop weights *recent* context heavily. The most recent tool result is what's most fresh in attention; the system prompt from 100 turns ago is at the back of the queue. Reinforcement keeps the goal *fresh* by re-presenting it after every tool call, riding alongside the tool result the agent is about to react to.

### Reinforcement in the wild

The pattern is already implicit in several major systems:

- **Claude Code's TODO list.** When you give Claude Code a multi-step task, it builds a TODO list and **re-renders that list in context after each tool call**. The effect is exactly reinforcement: the agent sees its plan and its progress every time it pauses to think. Armin called this out as the cleanest production example.
- **OpenAI Assistants' `additional_instructions`** — supports per-turn instruction injection.
- **LangGraph state machines** — node outputs include explicit `next_steps` annotations that get fed forward.
- **AutoGen's conversational message wrapping** — messages can be enriched with metadata.

### The "compact errors into context" connection

[12-Factor Agents Factor 9](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-09-compact-errors.md) — "Compact errors into context window" — is a specific instance of reinforcement: when a tool fails, don't just return the error verbatim. Summarize what failed, what was being attempted, what the agent might try next. The agent then has *enough* context to recover, without re-loading the full failed-trace into the window. Reinforcement generalizes this: every tool return (success or failure) is an opportunity to enrich context with reminders.

### Designing reinforcement payloads

Some heuristics that emerged in 2025-2026 practice:

- **Keep it short.** Reinforcement that's longer than the tool result it wraps becomes noise. Aim for 5-15% of the tool result's size.
- **Use structured format.** YAML/Markdown sections (`## TODO`, `## Objective`, `## Notes`) survive context-window pressure better than free-form prose.
- **Differential reinforcement.** Don't re-inject what hasn't changed. If the TODO didn't update, you can skip it; only re-inject when something actually shifted.
- **Don't reinforce in tight loops.** If the agent is making 10 fast tool calls per second to retrieve different chunks, reinforcement payloads dominate cost. Apply reinforcement at "decision points," not every leaf call.
- **Validate reinforcement empirically.** A/B test agents with and without reinforcement. Sometimes the added context hurts more than it helps if the model is already focused.

## Variants & related patterns

- [**Compaction**](compaction.md) — reduces history; reinforcement re-injects what compaction may have lost.
- [**Structured note-taking**](structured-note-taking.md) — the durable source-of-truth that reinforcement reads from.
- [**12-Factor Agents Factor 9**](#) — "compact errors into context" is a specific reinforcement pattern.
- **TODO-list patterns (Claude Code)** — the canonical implicit-reinforcement implementation.
- **Sub-agent architectures** (see [`../agt/`](#)) — sub-agents can be dedicated "plumbers" maintaining context for sibling agents.
- **Event-driven agent architectures** — context plumbing is the natural fit; MCP servers + webhook handlers + change feeds.

## When NOT to use

**Context plumbing as explicit discipline:**
- **Single-source agents** (chat-only assistants with no environment integration). Plumbing is unnecessary overhead.
- **Stateless / one-shot agents** — no context to plumb.

**Reinforcement:**
- **Very short agent runs** (<5 turns) — the original prompt is still fresh in attention; no need to re-inject.
- **Token-cost-sensitive deployments at very high QPS** — every tool call pays for reinforcement; can dominate.
- **When the model is reasoning-heavy** (o1, R1, Claude extended thinking) — the model is already actively recapping in its thoughts. Reinforcement can be redundant.
- **For deterministic workflow steps** (not agentic loops) — fixed code paths don't need reinforcement.
- **When you can't write a *concise* reinforcement payload.** Bloated reinforcement is worse than none.

## Implementations

| Pattern | Implementation options |
|---|---|
| **Context plumbing** | Event buses (Kafka, NATS), MCP servers exposing change feeds, scheduled summarizer agents, CDC pipelines, webhook handlers, Tailscale-mesh-of-agents (popular 2026 pattern), email/calendar/IoT MCP integrations |
| **Reinforcement (TODO style)** | Claude Code's built-in TODO list; LangGraph node outputs; AutoGen message wrappers; OpenAI Assistants `additional_instructions`; custom Python `with_context()` wrappers around tool returns |
| **Reinforcement (state-echo)** | Custom tool middleware that appends current state to every tool return |
| **Reinforcement (error compaction)** | 12-Factor Factor 9 patterns; standard in most production agent harnesses |

## Companies / projects

- **Anthropic** ✅ — Claude Code's TODO list is the canonical implicit reinforcement implementation ([reference](https://www.anthropic.com/engineering/claude-code-best-practices)).
- **HumanLayer (12-Factor Agents)** ✅ — Factor 9 codifies a reinforcement pattern.
- **Matt Webb / Acts Not Facts** ✅ — coined "context plumbing" ([source](https://interconnected.org/home/2025/11/29)).
- **Armin Ronacher (Sentry)** ✅ — coined "reinforcement" in the agent-loop sense ([source](https://lucumr.pocoo.org/2025/11/22/agent-design/)).
- **OpenAI Assistants API** ✅ — `additional_instructions` supports per-turn reinforcement.
- **LangChain / LangGraph** ✅ — explicit state forwarding between nodes.
- **Anthropic / Sentry / Linear / Notion** ⚠ — likely use plumbing-like architectures internally; not all publicly documented.

## Further reading

- [Context plumbing](https://interconnected.org/home/2025/11/29) — Matt Webb, Nov 2025 (the canonical source for "plumbing")
- [Agent design is still hard](https://lucumr.pocoo.org/2025/11/22/agent-design/) — Armin Ronacher, Nov 2025 (the canonical source for "reinforcement")
- [LLM APIs are a Synchronization Problem](https://lucumr.pocoo.org/2025/11/24/llm-sync/) — Armin Ronacher follow-up
- [12-Factor Agents Factor 9: Compact errors into context window](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-09-compact-errors.md) — Horthy
- [Claude Code best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices) — Apr 2025 (TODO-list reinforcement reference)
- [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic Sept 2025 (related framework)

---

*Diagram source: [`../diagrams/src/context-plumbing.d2`](../diagrams/src/context-plumbing.d2), [`../diagrams/src/reinforcement.d2`](../diagrams/src/reinforcement.d2)*
