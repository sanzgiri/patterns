# ReAct (Thought / Action / Observation)

**Aliases:** Reason+Act, Think-Act-Observe loop, the agentic loop, reasoning-and-acting prompting
**Category:** Foundations
**Sources:**
[Yao et al. — ReAct: Synergizing Reasoning and Acting in Language Models (Oct 2022, ICLR 2023)](https://arxiv.org/abs/2210.03629) ·
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[OpenAI — Function calling docs (2023-2026)](https://platform.openai.com/docs/guides/function-calling) ·
[LangChain ReAct agent docs](https://python.langchain.com/docs/integrations/tools)

---

## Problem

> [!TIP]
> **ELI5.** Bare LLMs can either *reason* (think out loud about a problem) or *act* (use a tool to look something up). ReAct is the deceptively simple idea: **let the model do BOTH, alternating**. The model writes a *Thought* about what to do next, *takes an Action* (calls a tool), reads the *Observation* (the tool's result), and loops. This pattern — invented in 2022 — is the foundation underneath essentially every agent you've heard of: ChatGPT browsing, Claude using tools, Cursor agent mode, Devin, every LangChain agent. Everything from a search-and-answer chatbot to a $50M autonomous coding agent is at its core some refinement of the Thought → Action → Observation loop.

ReAct is probably the most influential single agentic prompting pattern of the modern era. Yao et al. proposed it in October 2022 — *before* ChatGPT launched. The core insight was straightforward: prior work had shown LLMs could reason ([chain-of-thought](reasoning-prompts.md)) and that they could act (early tool use). ReAct combined them, letting the model *think* about what it knows, then *act* to learn more, then *think* about what to do with the new information, and so on.

By 2026, nearly every production agent traces its lineage back to ReAct, even if the framing has evolved. The [augmented LLM](augmented-llm.md) is the *unit*; ReAct is the *driver* — the loop that turns an augmented LLM into something that can multi-step toward goals. Modern variants (LangGraph agents, OpenAI Assistants, Claude with tool use, Anthropic computer use) all run a ReAct-like loop under the hood.

This page documents ReAct as the foundational pattern. Specific agent architectures that build on it have their own pages in `agt/`.

## How it works

> [!TIP]
> **ELI5.** Loop body: (1) **Thought** — model writes what it wants to do and why. (2) **Action** — model emits a tool call. (3) **Observation** — system runs the tool, feeds result back. Then loop back to Thought. The model decides on each iteration whether to do another action or to produce a final answer. That's it. That single loop is the foundation of virtually every modern agent.

![ReAct — Thought / Action / Observation loop](../diagrams/svg/react.svg)

### The canonical ReAct prompt structure

The original 2022 paper used a few-shot prompt with examples like:

```
Question: What is the elevation range for the area Colorado orogeny extends 
into in the eastern sector?

Thought 1: I need to search Colorado orogeny and find the area the eastern 
sector extends into.
Action 1: Search[Colorado orogeny]
Observation 1: The Colorado orogeny was an episode of mountain building in 
Colorado and surrounding areas...

Thought 2: It does not mention the eastern sector. I need to look up 
"eastern sector".
Action 2: Lookup[eastern sector]
Observation 2: The eastern sector extends into the High Plains...

Thought 3: The eastern sector extends into the High Plains. Now I need to 
find the elevation range of the High Plains.
Action 3: Search[High Plains]
Observation 3: High Plains refers to one of two distinct land regions...

[...continues until the agent finds the answer...]

Thought N: The elevation range is 1,800 to 7,000 ft.
Action N: Finish[1,800 to 7,000 ft]
```

The model is shown several examples like this in the prompt, then given a new question, and it follows the pattern: emits Thought, emits Action, system runs the tool, system appends Observation, model continues until it emits a `Finish[...]` action.

### Modern implementation

The 2022 paper used text-based action grammar (`Search[query]`, `Lookup[term]`, `Finish[answer]`). Modern implementations use [structured outputs / function calling](structured-outputs.md) — the actions are JSON-typed tool calls that the platform validates. But the *loop structure* is identical:

1. Model outputs reasoning (Thought) + tool call (Action) (potentially as one structured object).
2. Harness executes the tool call, produces an Observation.
3. Observation is appended to the conversation.
4. Loop.

Pseudocode (any modern framework):
```python
messages = [system_prompt, user_question]
while True:
    response = llm.complete(messages, tools=tool_schemas)
    if response.is_final_answer:
        return response.content
    tool_result = execute_tool(response.tool_call)
    messages.append(response)
    messages.append(tool_result)
```

This is what every modern agent framework runs at its core: a loop with one LLM call per iteration, possibly with one or more tool calls per iteration.

### Why ReAct works (mechanism)

ReAct works because it splits two distinct cognitive operations the LLM is doing:

- **Reasoning** is fully internal to the LLM — what it can derive from its weights and context. Good for synthesizing information, planning steps, drawing conclusions.
- **Acting** brings in fresh information — what the LLM cannot derive but can fetch (current data, computation results, side effects).

Without reasoning, the model can call tools but doesn't strategize between them. Without acting, the model can plan but has only training-data knowledge to work with. ReAct interleaves them, letting each support the other:
- Reasoning **selects** what to act on next.
- Acting **grounds** reasoning in fresh data.

The model on each turn doesn't have to decide everything; it just decides the next single action, with full context of what it's learned so far.

### What ReAct enables

The pattern unlocks:
- **Multi-hop retrieval.** Search, read, search again with what you learned. Foundational for sophisticated RAG.
- **Tool composition.** Chain calculator → web search → another calculator → API call.
- **Self-correction.** Observation tells the model the action failed; next thought decides a different approach.
- **Open-ended task execution.** "Book me a flight" requires many sub-tools; ReAct handles the orchestration.
- **Computer use.** Click → see new screen → think about new screen → next click. Computer-use agents are ReAct over GUI events.

### Limitations of vanilla ReAct

The 2022 ReAct paper had visible limitations that motivated later research:

- **Linear path; no backtracking.** The model can't say "I went the wrong way at step 4; let me re-do steps 4-6." Once it commits, it commits.
- **Context bloat.** Every Thought and every Observation accumulates in the context. Long trajectories blow the context window — leading to [context rot](../ctx/context-rot.md) and [compaction](../ctx/compaction.md) patterns.
- **Latency.** Each iteration is a round-trip LLM call. 20 iterations = 20 LLM calls.
- **Cost.** Same — many calls add up.
- **No reflection on whether it's making progress.** The model doesn't notice when it's stuck.

Modern variants address these:
- [**Reflection / self-critique**](reasoning-prompts.md) wraps a meta-loop around ReAct.
- [**Tree-of-Thought / planning agents**](reasoning-prompts.md) replace linear ReAct with search.
- [**Sub-agent architectures**](../agt/sub-agent-architectures.md) split work to keep individual contexts short.
- [**Compaction**](../ctx/compaction.md) and [**structured note-taking**](../ctx/structured-note-taking.md) manage context bloat.

### The 2026 view: ReAct is the substrate

By 2026, ReAct isn't usually written or discussed by name in product code — it's the implicit substrate. When you call an OpenAI Assistant, an Anthropic Claude with tools, or a LangGraph agent, the harness runs ReAct (or a variant) for you. You see "agent ran, used 7 tool calls, returned answer" — what happened underneath was ReAct.

This is similar to how programmers in 2026 don't usually write "for loops over arrays" — they write `.map()` and `.filter()` — but the for loops are still there, just abstracted. ReAct is still there.

### ReAct vs reasoning models

A 2024-2026 development: reasoning models (o1, Claude extended thinking, Gemini 2.5 thinking) have *internalized* the Thought step. Instead of emitting a visible Thought, the model thinks privately and then emits an Action.

This makes ReAct slightly less visible but doesn't replace it. The loop is still: think (internally now) → act → observe → think → act → ... The reasoning is hidden; the action loop is unchanged. Anthropic's tool-use with extended thinking, OpenAI's o-series with function calling — all run this slightly-modified ReAct loop.

For non-reasoning models, the explicit Thought is still valuable. For reasoning models, the Thought is implicit but the pattern continues.

### Common variants

- **Plan-and-execute.** First generate a plan (many steps), then execute each. Pre-commitment vs ReAct's step-by-step.
- **ReAct + Reflection.** After each action or at end, the model critiques and revises.
- **CodeAct / Program-aided.** The "Action" is generated code, not a tool call.
- **Multi-agent ReAct.** Multiple agents each running ReAct, communicating with each other.
- **Hierarchical ReAct.** Top-level ReAct loops decompose into sub-loops via sub-agents.
- **Computer-use ReAct.** Action is mouse/keyboard event; Observation is screen state.

All preserve the Thought/Action/Observation rhythm.

### Engineering details that matter

- **Tool result formatting** matters enormously. Pretty-printed errors vs raw stack traces, structured tool results, length limits on observations — all affect how well the next-iteration Thought can use the observation.
- **Cap iterations** to prevent runaway loops. Most production sets a max-iteration cap (50, 100) and fails gracefully.
- **Per-step timeout** for tools. The LLM is fast; tools are often slow. Per-tool timeouts prevent stuck iterations.
- **Visible vs hidden Thoughts** in user UI. For debugging visibility, show Thoughts; for clean UX, hide them.
- **Trace logging** every loop step. Essential for debugging when agents fail.

## Variants & related patterns

- [**Augmented LLM**](augmented-llm.md) — what ReAct loops over.
- [**Reasoning prompts**](reasoning-prompts.md) — the Thought step; family of techniques.
- [**Structured outputs**](structured-outputs.md) — how Actions are typed in practice.
- [**Single-agent with tools**](../agt/single-agent-with-tools.md) — the simplest production application of ReAct.
- [**Multi-agent orchestration**](../agt/multi-agent-orchestration.md) — composition of multiple ReAct loops.
- [**Sub-agent architectures**](../agt/sub-agent-architectures.md) — nested ReAct.
- [**Coding agents**](../agt/coding-agents.md) — ReAct specialized to code editing.
- [**Computer use**](../agt/computer-use.md) — ReAct specialized to GUIs.
- [**Compaction**](../ctx/compaction.md) — addresses context bloat from long ReAct trajectories.
- [**Maker-checker**](../agt/maker-checker.md) — wraps a verification step around the ReAct output.
- **Plan-and-execute** — variant that plans before acting.
- **CodeAct** — variant where actions are code.

## When NOT to use ReAct

- **Single-shot tasks** with no tool calls — ReAct overhead is wasted.
- **Pre-determined workflows** with known step sequences — use a workflow ([workflows vs agents](../agt/workflows-vs-agents.md)) instead.
- **Latency-critical paths** where loop overhead is unacceptable — pre-compute or use a static workflow.

For anything genuinely open-ended (the model has to decide what to do, the path varies by input), ReAct or a variant is essentially mandatory.

## Implementations

| Framework | ReAct support |
|---|---|
| **LangChain** | Native ReAct agent type; LangGraph for graph-based variants |
| **OpenAI Assistants API** | Built-in loop with tools |
| **OpenAI Agents SDK (2025)** | First-class agent primitive |
| **Anthropic SDK with tools** | Standard pattern in docs |
| **Vercel AI SDK** | `streamText` with tools loops automatically |
| **Mastra** | TS-first agent framework |
| **LlamaIndex** | ReActAgent |
| **Smolagents (Hugging Face)** | Minimalist ReAct framework |
| **AutoGPT, BabyAGI (2023)** | Early autonomous ReAct variants |
| **CrewAI, AutoGen** | Multi-agent ReAct |
| **Pydantic AI** | Type-safe ReAct |

## Companies / products built on ReAct (or variants)

Essentially every modern agent. Representative:

- **Anthropic** ✅ — Claude tool use, Claude Code, computer use all run ReAct-like loops.
- **OpenAI** ✅ — ChatGPT browsing, Assistants, Operator, Codex.
- **Google** ✅ — Gemini with function calling, Vertex AI Agents.
- **Cognition (Devin)** ⚠ — extended multi-hour ReAct.
- **Cursor, Aider, Cline, Continue, Replit Agent** ✅ — agent modes run ReAct.
- **GitHub Copilot Workspace** ✅ — plan-and-act variant of ReAct.
- **Perplexity, You.com, Phind** ✅ — retrieve + reason loops.
- **LangChain, LlamaIndex, AutoGPT and successors** ✅ — frameworks built around ReAct.

## Further reading

- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) — Yao et al. 2022 (the original paper)
- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024
- [LangChain ReAct agent guide](https://python.langchain.com/docs/concepts/agents/)
- [OpenAI function calling guide](https://platform.openai.com/docs/guides/function-calling)
- [The future of AI is agentic](https://huyenchip.com/2025/01/07/agents.html) — Chip Huyen overview
- [Anthropic — Tool use docs](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use)

---

*Diagram source: [`../diagrams/src/react.d2`](../diagrams/src/react.d2)*
