# The Agent Loop

**Aliases:** ReAct loop, think-act-observe loop, autonomous loop, agentic loop
**Category:** Agentic Patterns
**Sources:**
[Yao et al. — ReAct (2022)](https://arxiv.org/abs/2210.03629) ·
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[Lilian Weng — LLM Powered Autonomous Agents (2023)](https://lilianweng.github.io/posts/2023-06-23-agent/) ·
[Simon Willison — "An LLM in a loop with tools"](https://simonwillison.net/2025/Sep/18/agents/) ·
[Anthropic — Effective context engineering for AI agents (Sept 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

---

## Problem

> [!TIP]
> **ELI5.** You have an LLM. You want it to *do* things, not just *answer* — file a bug, refactor a function, book a flight, summarize 200 emails. A single prompt-and-response can't get there: it doesn't have permission to act, doesn't see results, can't course-correct. You need a *loop*: the LLM thinks about what to do, takes one step, sees what happened, thinks again. Like a human at a keyboard — except the keyboard is a set of tools.

A pure LLM call is a stateless function: text in → text out. To make it *do work* in the real world, you have to wrap it in a loop that:

1. Gives it tools (file I/O, web search, code execution, API calls, etc.).
2. Lets it choose which tool to invoke at each step.
3. Executes the chosen tool and feeds the result back in.
4. Keeps going until the LLM signals it's done.

This loop is the **agent**, and the loop's structure — what tools, what order, when to stop, what state survives between iterations — is the central design question of agentic AI in 2025-2026.

Simon Willison's definition (Sept 2025) is the cleanest currently in circulation:

> *"An LLM agent runs tools in a loop to achieve a goal."*

Six words. The rest of the field's literature is variations on what each of those words means in practice.

## How it works

> [!TIP]
> **ELI5.** Each iteration has three steps. **Think**: the LLM looks at the goal + what it's done so far + what it just learned, and decides what to do next. **Act**: it picks one tool and the arguments to call it with. **Observe**: the harness runs the tool, gets the result, and appends it to the conversation. Then back to think. Repeat until the LLM says "I'm done; here's the answer" — or until you cut it off.

![The agent loop — think, act, observe](../diagrams/svg/agent-loop.svg)

A concrete iteration looks like this. The agent's context holds (a) the system prompt, (b) the user's goal, (c) the tool definitions, (d) the conversation history so far including prior tool calls and their results, and (e) any [structured notes](../ctx/structured-note-taking.md) or reinforcement payloads. The agent harness calls the LLM with all of this.

The LLM responds with one of two things:
- **A tool call** — name + arguments — meaning "please do this for me, then ask me again." Modern providers (Anthropic, OpenAI, Google, Mistral) standardize this as native tool-calling: the response includes a structured `tool_use` block.
- **A final answer** — text content meaning "I'm done; here's the result." The harness ends the loop and returns this to the caller.

If it's a tool call, the harness executes the tool (calls a function, hits an API, runs `grep`, etc.) and packages the result as a `tool_result` message. That message gets appended to the conversation, and the loop iterates. Each turn the LLM sees one more `tool_use` / `tool_result` pair and decides anew.

The simplest pseudocode (this is essentially the whole pattern):

```python
def run_agent(goal, tools, model, max_iter=50):
    messages = [{"role": "user", "content": goal}]
    for _ in range(max_iter):
        response = model.call(messages, tools=tools)
        if response.stop_reason == "end_turn":
            return response.text          # ← agent says it's done
        for tool_use in response.tool_calls:
            result = execute_tool(tool_use)
            messages.append({"role": "assistant", "content": [tool_use]})
            messages.append({"role": "user",      "content": [tool_result(result)]})
    raise TimeoutError("agent exceeded max iterations")
```

That's it. Everything else in modern agent design — context engineering, sub-agents, evals, guardrails — is making this loop *robust enough to run for hours* without falling over.

### The three classic loop variants

Three named variants from the 2022-2023 literature are still the conceptual building blocks:

![Agent loop variants — ReAct, Plan-Execute, Reflexion](../diagrams/svg/agent-loop-variants.svg)

**ReAct** (Yao et al. 2022) — the canonical interleaved think/act/observe loop above. The default. Almost every production agent today is "ReAct + some additions."

**Plan-and-Execute** (popularized by LangChain in 2023) — the LLM plans *all* steps up front, then a (possibly cheaper) executor runs them in order. Faster and cheaper when the plan is reliable; rigid and brittle when the world disagrees mid-execution.

**Reflexion** (Shinn et al. 2023) — ReAct with an extra self-critique step on failure: after a wrong answer, the model writes a "lesson" (what went wrong, what to try next), the lesson goes into context, and the agent retries. Improved early benchmark scores; superseded in practice by stronger base models + structured note-taking + evals.

In 2025-2026 production, the dominant pattern is **modern ReAct**: a tool-using loop powered by a reasoning model (which thinks internally during each `think` step), enriched with [structured note-taking](../ctx/structured-note-taking.md), [compaction](../ctx/compaction.md) when context fills, [reinforcement](../ctx/context-plumbing-reinforcement.md) on each tool return, and explicit halting heuristics.

### What "step" really means in 2026

A 2023 ReAct "thought" was a literal `<thought>...</thought>` text block the model wrote out. With reasoning models (o1, o3, Claude extended thinking, DeepSeek-R1, Gemini Thinking), each *think* step now includes potentially **thousands of internal thinking tokens** the model generates before producing a tool call. Those thinking tokens are not free, but they dramatically improve the *quality* of each tool-choice decision — which often shrinks the total number of loop iterations needed.

This shifts the economics: an agent in 2024 might have done 30 cheap iterations to solve a task. The same agent in 2026 might do 8 expensive iterations and finish faster and more reliably. The loop shape is unchanged; the work-per-iteration shifted from "many small decisions" to "few well-considered decisions."

### When does the loop stop?

The hardest open question of agent design. Typical halt conditions, layered:

- **Natural stop** — the model produces a final answer / `end_turn`. Best case.
- **Goal-met check** — an explicit "is the task complete?" tool the agent must call to terminate. Common in coding agents.
- **Max iterations** — hard cap (e.g., 50 or 200 turns). Prevents runaway cost.
- **Max wall-clock time** — for human-facing agents, 60s, 5min, etc.
- **Token budget exceeded** — circuit-breaker on cumulative cost.
- **Repeated state** — agent has done the same thing 3 times in a row; likely stuck. Halt and escalate.
- **External cancellation** — user clicks Stop, parent agent decides to abandon a sub-agent.

A well-designed harness combines several. Just `max_iter` alone leads to agents that "almost finish" right at the cap; just "wait for natural stop" leads to runaway loops on poorly-specified tasks. Multi-layer halts + good [observability](#) of where the loop *actually* stopped is part of why production agents are harder than they look.

## Variants & related patterns

- [**Workflows vs Agents**](workflows-vs-agents.md) — when to use a fixed workflow vs an actual agent loop. Read this first if you're new.
- [**Single agent with tools**](single-agent-with-tools.md) — the default agent architecture; the loop with one specialist.
- [**Multi-agent orchestration**](multi-agent-orchestration.md) — multiple agents collaborating in nested or parallel loops.
- [**Sub-agent architectures**](sub-agent-architectures.md) — spawning fresh-context sub-agents from a parent loop.
- [**Coding agents**](coding-agents.md) — the loop applied to software development; the most-evolved specialty.
- [**Computer use**](computer-use.md) — the loop where the "tools" are mouse + keyboard.
- **ReAct paper** — original formulation.
- **Reflexion** — self-critique extension.
- **Tree-of-Thoughts / Graph-of-Thoughts** — branching-search variants of the loop.
- **Reasoning models** — the base capability that makes 2025-2026 loops work.
- [**Compaction**](../ctx/compaction.md), [**Reinforcement**](../ctx/context-plumbing-reinforcement.md), [**Structured note-taking**](../ctx/structured-note-taking.md) — context-engineering layers that keep long loops stable.

## When NOT to use

- **Single-turn tasks** — classification, extraction, translation, simple Q&A. A direct LLM call is faster, cheaper, and more reliable. Don't wrap a stateless task in a loop.
- **Fully deterministic workflows** — if you can write the steps as code with LLM calls embedded at known points, do that. Anthropic: *"Start with the simplest solution. Only add agentic systems when workflows are proven insufficient."*
- **Sub-second latency requirements** — each iteration is an LLM round-trip (often 1-10s with reasoning models). Agents are not real-time.
- **Highly regulated / audit-critical paths** — agent decisions can be hard to justify in compliance review. Use workflows so each decision is human-pre-approved.
- **Tight token budgets** — agents are token-hungry. A workflow with one carefully-prompted LLM call may do 5% of the work an agent would do at 1% of the cost.
- **When you can't define "done"** — if the goal can't be checked by the agent itself or a tool, agents tend to either stop too early or loop forever.

## Implementations

| Tool / framework | Loop style |
|---|---|
| **Anthropic Agent SDK** | Native tool-use loop with `tool_use` / `tool_result` primitives. The reference implementation. |
| **OpenAI Assistants API / Responses API** | Managed loop. `runs.create` polls until `completed` or tool-call required. |
| **OpenAI Codex CLI** | File/grep/run tools + ReAct loop, optimized for code. |
| **Claude Code** | Anthropic Agent SDK + filesystem/shell tools + TODO-list reinforcement. |
| **LangGraph** | Explicit state machine; loops are node→edge cycles. Most flexible / least magical. |
| **AutoGen** | `UserProxyAgent` + `AssistantAgent` exchanging messages in a loop. |
| **CrewAI** | Crew of agents with task assignment + sequential or hierarchical loops. |
| **smol-agents (HuggingFace)** | Minimal agent loop, "code-as-action" variant. |
| **Pydantic AI** | Type-safe tool-calling loop. |
| **Letta** | Loop with persistent memory blocks. |
| **DSPy** | Compiles loops as programs you can optimize. |

## Companies running agent loops in production

- **Anthropic** ✅ — Claude Code, Claude Cowork, Computer Use ([source](https://www.anthropic.com/research/building-effective-agents)).
- **OpenAI** ✅ — Codex CLI, Operator (computer-use agent), ChatGPT browsing/search, Deep Research.
- **Google DeepMind** ✅ — Gemini with tool use; agentic features in Gemini 2.x.
- **Cognition (Devin)** ✅ — long-horizon autonomous coding agent.
- **Cursor, Aider, Cline, OpenCode, Continue** ✅ — all major coding agents.
- **Perplexity, You.com** ✅ — research agents in production at consumer scale.
- **Sierra, Decagon, Lindy, Adept** ⚠ — customer-service and assistant agents; specifics vary.
- **Replit Agent, GitHub Copilot Workspace** ✅ — code-task agents in IDEs.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024 (the canonical practitioner essay)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) — Yao et al. 2022 (the original)
- [LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/) — Lilian Weng 2023 (the classic survey)
- [On the Definition of "Agents"](https://simonwillison.net/2025/Sep/18/agents/) — Simon Willison Sept 2025
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic Sept 2025
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366) — Shinn et al. 2023

---

*Diagram source: [`../diagrams/src/agent-loop.d2`](../diagrams/src/agent-loop.d2), [`../diagrams/src/agent-loop-variants.d2`](../diagrams/src/agent-loop-variants.d2)*
