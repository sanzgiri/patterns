# Workflows vs Agents

**Aliases:** orchestrated vs autonomous, choreographed vs agentic, structured vs open-ended
**Category:** Agentic Patterns
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[Simon Willison — On the definition of "agents" (Sept 2025)](https://simonwillison.net/2025/Sep/18/agents/) ·
[OpenAI — A practical guide to building agents (2025)](https://openai.com/) ·
[12-Factor Agents — Horthy](https://github.com/humanlayer/12-factor-agents)

---

## Problem

> [!TIP]
> **ELI5.** You have a job for an LLM. Two ways to organize it. **Workflow**: you (the engineer) write the steps — "first summarize, then classify, then dispatch." The LLM just executes each step you wrote. Predictable, cheap, easy to debug. **Agent**: you give the LLM a goal and tools, and *it* decides which step to take next. Flexible, but expensive, slower, and harder to debug. Many teams jump straight to "agent" because it sounds modern. Most should start with "workflow."

By 2024, the term "agent" was being applied to everything from a simple two-step LangChain pipeline to GPT-4 with the entire AWS console. The conceptual blur caused real problems: teams built complex multi-agent architectures for tasks a single prompt would solve, then complained that LLMs were "unreliable." Others built fixed workflows for inherently open-ended tasks and complained their system couldn't handle real users.

Anthropic's December 2024 essay ["Building effective agents"](https://www.anthropic.com/research/building-effective-agents) drew the line that has since become canonical:

> **Workflows** are systems where LLMs and tools are orchestrated through **predefined code paths**.
>
> **Agents** are systems where LLMs **dynamically direct their own processes** and tool usage, maintaining control over how they accomplish tasks.

This is the most-cited framing in current agentic literature. It clarifies that "agent vs workflow" is a *spectrum*, not a binary, and that the right choice depends on the task structure.

## How it works

> [!TIP]
> **ELI5.** Both have LLMs and tools. The difference is *who decides the order of operations*. In a workflow, the engineer hard-codes the order (with maybe a branch or two). In an agent, the LLM chooses the order on the fly, looking at each result to decide what to do next.

![Workflows vs Agents — who decides the path?](../diagrams/svg/workflows-vs-agents.svg)

A **workflow** looks like this in code: `step1 = summarize(text); step2 = classify(step1); step3 = dispatch(step2)`. The path is fixed. The LLM is called inside each step but doesn't choose what comes next. Branches are explicit `if` statements written by the engineer.

An **agent** looks like this: `while not done: action = llm.choose_tool(history); result = run(action); history.append(result)`. The LLM picks the action each time. The number of iterations isn't known up front. The same input can produce different paths on different runs.

The key trade-offs:

| Dimension | Workflow | Agent |
|---|---|---|
| **Cost** | Low — fixed number of LLM calls | High — many LLM calls per task |
| **Latency** | Low and predictable | High and variable |
| **Reliability** | High — each step pre-tested | Lower — emergent behavior |
| **Flexibility** | Low — only handles cases you coded for | High — adapts to novel inputs |
| **Debuggability** | Easy — linear trace | Hard — branching agent traces |
| **Best for** | Well-scoped, repeatable tasks | Open-ended, unpredictable tasks |
| **Failure mode** | Mishandles edge cases not coded for | Loops, hallucinates, expensive runs |

Anthropic's blunt advice: *"Start with the simplest solution and only add complexity when needed. Agentic systems often trade latency and cost for better task performance — and you should consider when this trade-off makes sense."*

### The five workflow building blocks

Before you reach for a full agent, there are five workflow patterns Anthropic documents that cover most production needs:

![Workflow building blocks](../diagrams/svg/workflow-blocks.svg)

**1. Augmented LLM.** A single LLM call enriched with retrieval, tools, and memory. The foundational unit; everything else composes from this. *Example:* a customer-support bot that retrieves the user's order history before answering.

**2. Prompt chaining.** Sequential LLM calls where each step's output feeds the next. Useful when a task decomposes cleanly into subtasks. Add programmatic gates (validation, structured-output checks) between steps. *Example:* generate an outline → write each section → polish the whole.

**3. Routing.** A classifier LLM dispatches inputs to one of several specialized downstream chains. Decouples concerns and lets each path be tuned independently. *Example:* customer message → route to {refund, technical, billing, escalation} pipeline.

**4. Parallelization.** Many LLM calls in parallel, then aggregate. Two flavors: **sectioning** (break a task into independent chunks and process in parallel) and **voting** (run the same task N times and aggregate). Useful for speed or for robustness via consensus. *Example:* code review where one LLM checks security, another checks performance, another checks style — in parallel.

**5. Orchestrator-workers.** A central LLM plans subtasks dynamically and delegates each to a worker LLM. Closer to agentic — but still typically one round of planning then parallel execution, not a full loop. *Example:* "research these 10 companies" → orchestrator decides what to look up for each → workers fetch in parallel.

**6. Evaluator-optimizer.** A generator-LLM produces output, an evaluator-LLM critiques it, the loop continues until the evaluator is satisfied. Often classified as a workflow because the loop structure is fixed, even though it *iterates*. Useful when there's a clear quality criterion. *Example:* literary translation that improves until idioms read naturally.

Only when none of these five fit cleanly should you reach for a full **agent loop** (covered in [`agent-loop.md`](agent-loop.md)).

### When agents earn their cost

Anthropic identifies the cases where agents — despite the cost and latency penalty — genuinely outperform workflows:

- **Tasks with unpredictable structure** — when you can't enumerate the steps in advance.
- **Tasks requiring many iterations** of trial and error (coding, debugging, research).
- **Tasks with branching dependent on intermediate results** that you couldn't pre-route.
- **Long-horizon work** spanning many tool calls (more than ~5-10 distinct decisions).
- **Tasks where the cost of an LLM call is small** compared to the cost of a wrong outcome.

Coding agents (Claude Code, Devin, Cursor's agent mode) sit firmly in the "earn it" zone: a multi-hour refactor genuinely can't be pre-scripted, the search-space is huge, the value of getting it right is high, and the developer can verify the output. Customer-service "agents" usually sit in the workflow zone — most tickets follow one of a small number of paths, and over-engineering them adds latency without quality gains.

### The 2025-2026 hybrid pattern

In practice, most production "agent" systems are **mostly workflow with agentic islands**. A system might be:

```
fixed pipeline:
  → ingest user message
  → classify intent (workflow: routing)
  → if "complex research":
      spawn research agent (agent loop, ~10-30 turns)
  → format response
  → return to user
```

The workflow handles the predictable shell; an agent handles the one open-ended subtask. This pattern — sometimes called "agent as a function" or "scoped agent" — keeps cost and latency under control while still benefiting from open-ended autonomy where needed.

[12-Factor Agents](https://github.com/humanlayer/12-factor-agents) (Horthy 2024) makes a stronger version of this argument: "many things being branded as agents are mostly deterministic software with an LLM sprinkled in at strategic points." That's not a critique — it's the recipe that works in production.

### Choosing between them — the decision checklist

Use this in order. If you can stop at any step, stop:

1. **Can a single LLM call do it?** → Use a single LLM call (augmented LLM).
2. **Does it decompose into a fixed sequence?** → Prompt chaining.
3. **Does it branch on input type?** → Routing.
4. **Can subtasks run independently?** → Parallelization.
5. **Need iterative refinement against a clear criterion?** → Evaluator-optimizer.
6. **Need dynamic planning of subtasks?** → Orchestrator-workers.
7. **Genuinely needs open-ended, multi-step exploration?** → Agent loop.

Most teams find they stop at step 1, 2, or 3 for the bulk of their use cases — and reach for step 7 only for narrow, high-value flows.

## Variants & related patterns

- [**Agent loop**](agent-loop.md) — what step 7 actually looks like.
- [**Single agent with tools**](single-agent-with-tools.md) — the simplest agent variant; often the right one.
- [**Multi-agent orchestration**](multi-agent-orchestration.md) — when one agent isn't enough; rarely the right starting point.
- **Augmented LLM, Prompt Chaining, Routing, Parallelization, Orchestrator-Workers, Evaluator-Optimizer** — the five (six) workflow patterns enumerated above; future pages in `../wf/` cover each in depth.
- [**12-Factor Agents**](#) (Horthy) — manifesto arguing most "agents" should be workflows.
- **LangGraph** — the framework explicitly designed for workflows-with-agentic-nodes.
- **DSPy** — workflows you can optimize as programs.

## When NOT to use this distinction

- **In casual writing.** "Agent" has become the popular term; pedantically correcting people is rarely productive. Save the workflow-vs-agent precision for *design* conversations.
- **When the system genuinely does both.** Many real systems are workflows-with-agents and the distinction blurs.
- **For benchmark comparisons.** SWE-bench, GAIA, AgentBench etc. test "agentic" capability — pure workflows can't compete, by definition. The question isn't which is "better"; it's which fits.

## Implementations

| Tool / framework | Default leaning | Notes |
|---|---|---|
| **LangChain Expression Language (LCEL)** | Workflow | Chains, runnables, fixed graphs. |
| **LangGraph** | Both | Explicit state machine — express workflows or agent loops or mixed. |
| **DSPy** | Workflow | Programs that compile and optimize. |
| **Pydantic Graph** | Both | Typed nodes; structured workflows + loops. |
| **OpenAI Assistants API** | Agent (managed loop) | Hard to do pure workflow inside it. |
| **Anthropic Agent SDK** | Agent | Workflow possible by limiting tools / iteration count. |
| **CrewAI** | Agent (multi-agent) | Optimized for crew-style orchestration. |
| **AutoGen** | Agent | Multi-agent conversation as the primitive. |
| **n8n, Make, Zapier (+ LLM nodes)** | Workflow | Visual workflows with LLM steps inserted. |
| **Step Functions / Airflow / Prefect (+ LLM tasks)** | Workflow | Production-grade workflow engines hosting LLM tasks. |

## Companies / projects making the distinction explicit

- **Anthropic** ✅ — coined the canonical framing ([source](https://www.anthropic.com/research/building-effective-agents)).
- **OpenAI** ✅ — "A practical guide to building agents" (2025) makes a similar distinction.
- **HumanLayer (12-Factor Agents)** ✅ — argues most "agents" should be workflows.
- **Simon Willison** ✅ — public commentary curating definitions (Sept 2025 post).
- **LangChain** ✅ — entire LangGraph product reflects the workflow/agent spectrum.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024 (the source)
- [On the definition of "agents"](https://simonwillison.net/2025/Sep/18/agents/) — Simon Willison Sept 2025 (survey of competing definitions)
- [A practical guide to building agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) — OpenAI 2025
- [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) — Horthy (the workflow-leaning manifesto)
- [Agent vs Workflow — LangChain blog](https://blog.langchain.dev/) — multiple posts contrasting the two

---

*Diagram source: [`../diagrams/src/workflows-vs-agents.d2`](../diagrams/src/workflows-vs-agents.d2), [`../diagrams/src/workflow-blocks.d2`](../diagrams/src/workflow-blocks.d2)*
