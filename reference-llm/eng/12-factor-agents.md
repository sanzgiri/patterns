# 12-Factor Agents

**Aliases:** 12FA, Dex Horthy methodology, principled agent design, agent control plane
**Category:** Production Engineering
**Sources:**
[Dex Horthy — 12-Factor Agents (2024-2025)](https://github.com/humanlayer/12-factor-agents) (canonical repo; 11k+ stars) ·
[HumanLayer blog posts on agent reliability](https://humanlayer.dev/blog) ·
inspired by [Adam Wiggins — The Twelve-Factor App (2011)](https://12factor.net/) ·
echoed by Anthropic's [Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents), Hamel Husain, Eugene Yan production playbooks

---

## Problem

> [!TIP]
> **ELI5.** Most agent frameworks abstract away the wrong things. They give you "just write a prompt and we'll handle everything!" — until something goes wrong, you can't tell why, you can't fix it without re-learning the framework, and you can't customize the pieces you actually need. **12-Factor Agents** is Dex Horthy's distillation (from years at HumanLayer building production LLM apps) of the *non-negotiable* principles: own your prompts, own your context window, own your control flow, treat agents as stateless reducers, separate human contact from tool calls. Modeled after the famous 12-Factor App methodology for cloud-native software. The big idea: production LLM software is *mostly traditional software with a probabilistic core*. The factors specify which pieces you must control directly vs. delegate to a framework.

The original [12-Factor App](https://12factor.net/) (Adam Wiggins, Heroku, 2011) was the manifesto for cloud-native software. It told engineers what they had to get right (config in environment, stateless processes, port binding) regardless of language or platform. It's still the most-cited app design document of the cloud era.

12-Factor Agents takes the same posture for LLM-powered software. Horthy's argument (made in blog posts, talks, and the [github.com/humanlayer/12-factor-agents](https://github.com/humanlayer/12-factor-agents) repo): the "framework-magic" approach to agents fails at production scale. The successful production agents he's seen and built share a small set of principles that the popular frameworks often *obscure*.

The work resonated. The repo grew to 11k+ stars; the principles get cited in Anthropic's [Building effective agents](https://www.anthropic.com/research/building-effective-agents), Eugene Yan's posts, Hamel Husain's evals work, and broader practitioner discussion.

## How it works

> [!TIP]
> **ELI5.** Twelve principles. The pattern: name what you *must own yourself* in a production agent, and what shape it should take. Half are about *what you own* (prompts, context, control flow). Half are about *architectural choices* (stateless reducers, small focused agents, owning your tools as plain functions). Treat anything the framework hides as a candidate to pull out.

![12-Factor Agents](../diagrams/svg/12-factor-agents.svg)

### The 12 factors (Horthy's framing)

The factors evolved slightly across versions of the repo; here's the late-2024 / 2025 canonical list:

**1. Natural language to tool calls.** The fundamental atomic unit of an agent is: take natural language, produce a structured tool call. Build everything around that primitive.

**2. Own your prompts.** Don't let frameworks templatize them away. Your prompts are core IP. Write them, version them, test them, iterate them yourself.

**3. Own your context window.** Decide what goes in. Don't let the framework decide to silently insert a system prompt, RAG results, or anything else. Every token in the context is your decision.

**4. Tools are just structured outputs.** A "tool" isn't a magical framework abstraction. It's a structured output schema. When the LLM emits a tool call, you parse the structured output and dispatch. Avoid frameworks that wrap this in unnecessary layers.

**5. Unify execution state and business state.** Don't have a parallel "agent state" tracked by the framework AND your business state in your DB. Use one. Most production patterns: business state in DB, agent state derived from it.

**6. Launch, pause, resume with simple APIs.** Agent runs should be operable like normal job runs: explicit start, explicit pause (and resume), explicit completion. Not "the agent decides when it's done."

**7. Contact humans with tool calls.** When the agent needs a human (approval, clarification, escalation), express that as a tool call. The agent's "ask for human review" is the same shape as its "call an API." Both are structured outputs to a router.

**8. Own your control flow.** Don't let the LLM decide whether to loop, branch, retry. Express control flow in code. The LLM emits decisions; code executes them.

**9. Compact errors into context.** When something fails, summarize it into the context for the next turn (don't dump full stack traces). The model needs a short, useful description of what went wrong.

**10. Small, focused agents.** Many small agents (or single agent + many skills) beats one mega-agent that does everything. Each agent has a narrow job, well-defined inputs and outputs.

**11. Trigger from anywhere; meet users where they are.** Agents should be invokable from Slack, email, CLI, web, API — same agent, multiple entry points. Don't build a "chat UI agent" that's tied to one interface.

**12. Make your agent a stateless reducer.** This is the deepest factor. The agent function should be: `state, message -> new_state, response`. Pure. Reproducible. Replayable. State persistence is the harness's job, not the agent's.

(The 12-Factor App original was: codebase, dependencies, config, backing services, build/release/run, processes, port binding, concurrency, disposability, dev/prod parity, logs, admin processes. The parallel is loose but recognizable.)

### Why these factors (the failure modes they prevent)

Each factor corresponds to a real production failure Horthy and others encountered:

- **Frameworks hiding prompts** → you can't debug or improve them.
- **Frameworks hiding context window** → you don't know what the model actually saw.
- **Frameworks tracking parallel state** → out-of-sync state, mysterious bugs.
- **Free-running control flow** → unbounded loops, infinite retries, runaway cost.
- **Mega-agents** → unpredictable, untestable, hard to evolve.
- **Stateful agents** → un-replayable, hard to debug, hard to scale.

The factors aren't theoretical — they're *post-mortem-driven*. They map to specific outages and quality problems in production deployments.

### How this maps to Anthropic's Building effective agents

Anthropic's December 2024 post [Building effective agents](https://www.anthropic.com/research/building-effective-agents) doesn't cite 12-Factor explicitly but echoes much of it:

- "Start simple" → focus on small agents (Factor 10)
- "Don't use frameworks for the core agent loop" → own your prompts, context, control flow (Factors 2, 3, 8)
- "Composable patterns" → workflows + agents as primitives, not monolithic abstractions (Factors 4, 10)
- "Maintain transparency" → traceable execution state (Factor 12)

The convergence isn't accidental. Both works are reactions to the same period of framework-heavy agent libraries (LangChain v0 era) that hid too much.

### Practical application

When evaluating an agent framework or designing your own:

1. **Can you write the prompts in plain text and version them?** (Factor 2)
2. **Can you inspect every token sent to the model?** (Factor 3)
3. **Is the control flow written in your code, not the LLM's brain?** (Factor 8)
4. **Can you replay a session deterministically given the same state?** (Factor 12)
5. **Are tools just functions with typed inputs/outputs?** (Factor 4)
6. **Can a human intervene mid-run via a structured signal?** (Factor 7)

If any answer is "no" or "sort of, but the framework does it for me," that's a red flag. The factor isn't being followed.

### What 12-Factor does NOT say

- It doesn't pick a programming language.
- It doesn't pick a framework (it's framework-skeptical generally).
- It doesn't prescribe a specific architecture (workflow vs agent vs swarm).
- It doesn't tell you what tools to use (LangSmith vs Braintrust, etc.).

It's a principles document, not an implementation guide. Multiple architectures satisfy the principles.

### Where Horthy's view diverges from mainstream advice

A few opinions not universally held:

- **Skepticism of agent frameworks.** Horthy explicitly recommends building your own harness for production systems. Most newcomers benefit from a framework first.
- **Strong preference for "control flow in code, decisions from LLM"** over "LLM does both." Some agentic workflows (especially [ReAct](../fnd/react.md)) explicitly mix the two.
- **Skepticism of multi-agent.** Horthy argues small focused agents > complex multi-agent orchestration. [Anthropic's multi-agent post](https://www.anthropic.com/engineering/built-multi-agent-research-system) shows multi-agent has its place; the disagreement is about when.
- **Emphasis on stateless reducers** is more functional / Elm-influenced than typical agent practice; trade-offs apply.

These divergences are honest disagreements among practitioners, not settled facts.

### Anti-patterns (factor violations)

- **Framework-managed prompts**: you can't read the system prompt without diving into framework source.
- **Hidden RAG**: the framework retrieves and injects without your knowing what or when.
- **Stateful agents**: you can't tell what state the agent is in without running it.
- **Tools wrapped in abstraction**: a "Tool" object that hides what's actually called.
- **Free-running while loops**: the LLM decides when to stop; sometimes it doesn't.
- **Mega-prompts**: one 10K-token prompt that handles 17 different intents.
- **No replay**: you can't reproduce yesterday's incident.
- **Chat-only entry point**: agent tied to one UI; can't run from CI or batch.

### Production engineering implications

Following 12-Factor leads to a particular architecture:

- **Thin LLM call**: receive context + decisions; return structured output. Stateless function.
- **Fat harness**: assembles context, calls LLM, dispatches tools, persists state, handles errors. This is your code, your design.
- **Plain text prompts**: stored in files, versioned in git, eval'd separately.
- **Explicit control flow**: loops, conditionals, retries are code constructs, not LLM behaviors.
- **Replayable runs**: every run logs full state; can be re-executed.
- **Small modular pieces**: composable; each testable in isolation.

This is the architecture Anthropic, OpenAI, and most successful production deployments converge on.

## Variants & related patterns

- [**Harness design**](harness-design.md) — the code shape 12-Factor implies.
- [**Brain vs hands**](brain-vs-hands.md) — model split is a 12-Factor-compatible architecture.
- [**Augmented LLM**](../fnd/augmented-llm.md) — Anthropic's atom; 12-Factor at the prompt level.
- [**Workflows vs agents**](../agt/workflows-vs-agents.md) — 12-Factor leans workflow.
- [**Single agent with tools**](../agt/single-agent-with-tools.md) — typical 12-Factor shape.
- [**Maker-checker**](../agt/maker-checker.md) — "contact humans with tool calls" (Factor 7).
- [**LLM observability**](llm-observability.md) — Factor 12 demands strong observability.
- [**Rainbow deployments**](rainbow-deployments.md) — 12-Factor compatible deployment strategy.
- [**Eval-driven development**](../qua/eval-driven-development.md) — required to evolve prompts (Factor 2).
- **12-Factor App** (Wiggins 2011) — the original inspiration.

## When NOT to use

- **Prototypes and demos.** Premature 12-Factor adoption slows learning.
- **One-off scripts.** Overhead exceeds benefit.
- **Research code.** Adherence isn't the goal.
- **Heavily framework-locked deployments** where rewriting isn't an option.

## Implementations

| Framework | 12-Factor friendliness |
|---|---|
| **Vercel AI SDK** | High — primitives are small and composable |
| **Mastra** | High — explicit primitives, no hidden magic |
| **LangGraph** | Medium — gives control but you can still hide things |
| **Pydantic AI** | High — type-safe, explicit |
| **OpenAI Agents SDK (2025)** | Medium-high — improved over older Assistants |
| **DSPy** | Medium — programmatic, but compilation hides some details |
| **HumanLayer** | Native — Horthy's own platform |
| **LangChain (v0)** | Low — hides too much; v0.3+ improved |
| **AutoGen** | Medium — multi-agent abstractions can hide flow |
| **Custom harness** | Variable — easy to violate without discipline |

## Companies / products following 12-Factor (in practice or in principle)

- **HumanLayer** ✅ — Horthy's own platform, built around the principles.
- **Anthropic** ⚠ — Building effective agents echoes much of 12-Factor.
- **OpenAI** ⚠ — Agents SDK design reflects similar principles.
- **Eugene Yan, Hamel Husain practices** ⚠ — broadly compatible.
- **Most successful production LLM teams** ⚠ — converge here regardless of source.
- **Cursor, Continue** ⚠ — own their prompts and control flow.
- **Glean, Vectara, Pinecone Assistant** ⚠ — productize 12-Factor-aligned shapes.

## Further reading

- [12-Factor Agents GitHub](https://github.com/humanlayer/12-factor-agents) — Horthy's canonical repo
- [HumanLayer blog](https://humanlayer.dev/blog) — extended posts on the principles
- [The Twelve-Factor App](https://12factor.net/) — Adam Wiggins, 2011 (original)
- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [What we learned from a year of building with LLMs](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — Yan, Bischoff, Shankar et al.
- [Hamel Husain's evals posts](https://hamel.dev/blog/)

---

*Diagram source: [`../diagrams/src/12-factor-agents.d2`](../diagrams/src/12-factor-agents.d2)*
