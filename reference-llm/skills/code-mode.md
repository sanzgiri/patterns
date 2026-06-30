# Code Mode (Code Execution with MCP)

**Aliases:** Code Execution with MCP, agent code generation, programmatic tool orchestration, sandboxed code agents
**Category:** Skills & Packaging
**Sources:**
[Anthropic — Code execution with MCP (Nov 2025)](https://www.anthropic.com/engineering/code-execution-with-mcp) ·
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[OpenAI Code Interpreter / Assistants v2](https://platform.openai.com/docs/assistants/tools/code-interpreter) ·
[E2B sandbox documentation](https://e2b.dev/docs) ·
related: smolagents (HuggingFace 2024-2025), Cursor agent mode, Replit Agent

---

## Problem

> [!TIP]
> **ELI5.** Tool-using agents traditionally work turn by turn: "call tool A → wait → see result → call tool B → wait → see result → ..." For a task needing 10 tool calls, that's 10 LLM round-trips, 10 prompts with growing history, lots of cost and latency. **Code Mode** flips it: the agent emits **a single program** (Python or TypeScript) that calls all the tools in sequence, runs it in a sandbox, and returns the result in ONE LLM call. The tools are wrapped as functions in the sandbox's runtime; loops, conditionals, and error handling happen *in code*, not via additional LLM turns. Anthropic's November 2025 post on **Code Execution with MCP** made this the canonical pattern for MCP-rich agents. The result: roughly 3× speed-up and 10× cost reduction on multi-step tool-heavy work, plus dramatically better orchestration of complex workflows (try-except, retries, loops, map-over-list — all natural in code, awkward in turn-by-turn tool use).

The problem with turn-by-turn tool use compounds as tasks get more tool-heavy. Each tool call requires:
- A full LLM call with the entire conversation history
- Parsing the model's tool-call output
- Executing the tool
- Constructing a new prompt with the tool result
- Another LLM call to decide the next step

For a task that needs 20 tool calls, that's 20 LLM calls. The conversation history grows linearly, costing more each turn. And the LLM has to "remember" what it's trying to do across all those turns, which interacts badly with [context rot](../ctx/context-rot.md).

[Anthropic's November 2025 post](https://www.anthropic.com/engineering/code-execution-with-mcp) made the alternative concrete: agents that emit code orchestrating MCP tools, not agents that emit individual tool calls. The pattern was foreshadowed by HuggingFace smolagents (2024), Cursor's code-mode features, OpenAI's Code Interpreter, and various academic work — but Anthropic's MCP-centered framing crystallized the production pattern.

## How it works

> [!TIP]
> **ELI5.** Set up a sandbox (E2B, Modal, Pyodide, Docker) where the agent can run code. Wrap each MCP tool as a function in the sandbox's runtime. The agent gets a prompt like "Here are the functions you can call. Write a program that solves the user's task." The agent emits a Python (or TypeScript) script. The harness runs it; captures stdout, stderr, return value; returns to the agent. Often the task is done in one round-trip. For complex tasks, the agent may iterate — but each iteration does much more work per LLM call.

![Code Mode](../diagrams/svg/code-mode.svg)

### The mechanics

1. **Sandbox setup.** A secure sandboxed runtime (E2B, Modal, Pyodide for browser, Docker for self-host) where arbitrary code can execute safely.
2. **MCP tools wrapped as functions.** For each MCP tool exposed to the agent, generate a sandbox-side function stub. Calling the function in the sandbox triggers an MCP RPC back to the host.
3. **Agent receives an enhanced prompt.** Instead of "you can call these tools," it's "you have these Python functions; write a script to accomplish the task."
4. **Agent emits code.** The model writes a complete program — imports, loops, error handling, prints.
5. **Harness executes the code.** Captures stdout/stderr, return values, error tracebacks.
6. **Result returns to the agent.** If the task is complete, return the program output to the user. If not (error, ambiguous result, follow-up needed), the agent iterates with the output as feedback.

### A concrete example

User task: "Find all `.py` files in the project, count the total lines of code, and identify any file over 500 lines."

**Turn-by-turn tool use (10+ LLM calls):**
```
Turn 1: list_files('.', pattern='*.py') → ['a.py', 'b.py', 'c.py', ...]
Turn 2: read_file('a.py') → '200 lines'
Turn 3: read_file('b.py') → '150 lines'
...
Turn N: synthesize answer
```

**Code mode (1 LLM call):**
```python
files = list_files('.', pattern='*.py')
sizes = {f: count_lines(f) for f in files}
total = sum(sizes.values())
oversized = [f for f, n in sizes.items() if n > 500]
print(f"Total LOC: {total}")
print(f"Files >500 lines: {oversized}")
```

The agent writes the loop and the threshold check in code, executes once, gets the answer.

### Why it's so much more efficient

Three multipliers:

**(a) Fewer round-trips.** N tool calls become 1 LLM call + 1 code execution. Wall-clock latency drops by N-1× round-trip times.

**(b) No context growth per tool call.** Turn-by-turn, every tool call adds to the prompt (the previous tool calls + results). Code mode condenses all of that into one program + one captured output.

**(c) Better orchestration.** Loops, conditionals, retries, error handling, map-over-collection, parallel calls — natural in code, awkward (or impossible) in turn-by-turn tool use.

The trade-off: the agent needs to be able to write code (which frontier models are *very* good at). And you need the sandbox infrastructure.

### What changed with MCP

Before MCP, code-mode agents could call only the tools the harness pre-wrapped. With MCP:

- Any MCP server's tools are auto-discoverable.
- Wrappers can be generated automatically from MCP tool schemas.
- An agent can use tools from **multiple MCP servers** in a single program.
- The sandbox becomes a *universal MCP orchestrator*.

The Nov 2025 Anthropic post emphasized this: code mode + MCP unlocks the full ecosystem of MCP servers (file system, GitHub, Slack, Notion, custom internal tools, etc.) through a single coherent execution model.

### When code mode shines

- **Multi-step data tasks**: analyze logs, transform files, aggregate from multiple sources.
- **Codebase work**: read many files, search, refactor, run tests.
- **API orchestration**: chain multiple API calls with logic between them.
- **Math / quantitative**: code does math; LLM does reasoning.
- **Bulk operations**: map a transformation over a list.
- **Error recovery**: try/except in code is robust; turn-by-turn retry logic is fragile.
- **Long-running tasks**: code can checkpoint, retry, stream progress.

### When code mode doesn't fit

- **Single tool call** tasks (the overhead of code generation isn't worth it for one call).
- **Highly interactive tasks** where each step needs user feedback before continuing.
- **Tasks where the model can't write code well** (very specialized domains).
- **Environments without a sandbox** (you really shouldn't run agent-written code without one).
- **Tasks where every intermediate step must be visible to the user.** Code runs in one go; intermediate state is harder to expose.

### The sandbox landscape (2025-2026)

| Sandbox | Type | Use case |
|---|---|---|
| **E2B** | Cloud Linux VMs, fast cold start | General code agents; ~150ms cold-start |
| **Modal** | Cloud serverless containers | Heavy compute, GPUs |
| **Daytona** | Cloud dev environments | Long-lived dev environments |
| **Pyodide** | Python-in-browser | Client-side, no server needed |
| **Docker** | Self-hosted containers | Enterprise, on-prem |
| **WebContainers** (StackBlitz) | Browser Node.js | Browser-native JS code |
| **Vercel Sandbox** | Vercel-hosted | Within Vercel deployments |
| **Anthropic Code Execution tool** | Anthropic-managed | Native Claude tool (built-in code execution) |

Different trade-offs on cold-start latency, supported languages, supported packages, cost, and security boundaries.

### Security model

Running agent-written code requires careful sandboxing — this is the deeper version of the [containment / blast radius](../sec/containment-blast-radius.md) discipline:

- **No host network access** unless explicitly granted (see [egress allowlisting](../sec/egress-allowlisting.md)).
- **No host filesystem access** unless explicitly mounted.
- **Resource limits** (CPU, memory, wall-time, disk).
- **Ephemeral state** — sandbox destroyed after task.
- **Output capture** — stdout/stderr captured, not returned raw to user.
- **Network egress** — outbound to APIs only via approved hosts.

E2B and similar sandbox-as-a-service products handle these defaults. Self-rolled sandboxes need to enforce them explicitly.

### Engineering details

- **Tool wrapper generation.** Auto-generate sandbox-side stubs from MCP tool schemas. Anthropic publishes generators.
- **Output capture.** Capture stdout, stderr, return value, raised exceptions. Provide all three to the agent on follow-up.
- **Streaming output** (optional). Stream stdout to the user while code runs for long tasks.
- **Timeout per execution.** Default 30-60s; longer with explicit opt-in.
- **Per-call cost telemetry.** Track tokens for code emission + sandbox compute cost.
- **Error feedback.** When code throws, include the traceback in the next prompt; the agent debugs.
- **Iterative refinement.** When the first program doesn't work, the agent often debugs in 1-2 follow-up turns — still far fewer than the turn-by-turn equivalent.
- **Logging.** Code mode produces clear audit trails (the code IS the plan).

### Anti-patterns

- **No sandbox.** Running agent code on the host is the easiest way to disaster.
- **Unlimited sandbox lifetime.** Leaks resources; gives attackers persistence.
- **Sharing sandbox state across users.** Cross-tenant leak risk.
- **Code mode for trivial tasks.** Overhead exceeds benefit; use direct tool calls.
- **Auto-installing arbitrary packages.** Supply-chain attack vector.
- **No timeout.** Stuck programs hang the agent.
- **Returning raw stderr to user.** May leak sensitive info from tool errors.
- **No tool whitelisting in the sandbox.** Agent can call anything; need to enforce.

### Composition with other patterns

- **Code mode inside [orchestrator-workers](../wf/orchestrator-workers.md)**: orchestrator dispatches workers; each worker may execute code.
- **Code mode + [skills](skill-md-format.md)**: a skill can ship a program template; the agent fills in specifics and executes.
- **Code mode + [MCP](#)**: code mode is the canonical way to use rich MCP ecosystems.
- **Code mode + [evaluator-optimizer](../wf/evaluator-optimizer.md)**: tests in the sandbox are the evaluator.
- **Code mode + [maker-checker](../agt/maker-checker.md)**: the program is the proposed action; review before execution.

## Variants & related patterns

- [**SKILL.md format**](skill-md-format.md) — skills often package code mode tasks.
- [**AGENTS.md**](agents-md.md) — project-level standing instructions that code mode reads.
- [**Single agent with tools**](../agt/single-agent-with-tools.md) — code mode is a tool use pattern.
- [**Coding agents**](../agt/coding-agents.md) — most modern coding agents use code mode.
- [**Computer use**](../agt/computer-use.md) — the more open-ended cousin.
- [**Workflows vs agents**](../agt/workflows-vs-agents.md) — code mode is often workflow execution.
- [**Containment & blast radius**](../sec/containment-blast-radius.md) — security model.
- [**Browser as sandbox**](../sec/browser-as-sandbox.md) — sandbox-execution principle.
- **MCP** (`proto/mcp`) — the underlying tool protocol.
- **smolagents** (HuggingFace) — code-mode agent library.

## When NOT to use

- **Tasks with one tool call** — overhead exceeds benefit.
- **Tasks needing per-step user approval** — code runs all at once.
- **Without proper sandboxing** — too dangerous.
- **Domains the model can't code well in.**
- **Latency-critical single calls** where sandbox cold start hurts.

## Implementations

| Tool | Code Mode support |
|---|---|
| **Anthropic Claude (Code Execution tool)** | Built-in tool (2024-2025) |
| **OpenAI Code Interpreter / Assistants** | Built-in tool |
| **HuggingFace smolagents** | First-class code-mode agent library |
| **E2B SDK** | Sandbox + MCP integration |
| **Modal SDK** | Sandbox for code agents |
| **Cursor agent mode** | Internal code-mode for refactoring |
| **Replit Agent** | Code-mode by default |
| **LangChain / LangGraph** | `PythonREPLTool` and integrations |
| **Vercel AI SDK + Vercel Sandbox** | Code-mode primitive |
| **Anthropic MCP servers** | Auto-wrappable for code-mode |
| **Pyodide / WebContainers** | Browser-side code mode |

## Companies / products built on code mode

- **Anthropic** ✅ — recommends it for MCP-rich agents ([Nov 2025 post](https://www.anthropic.com/engineering/code-execution-with-mcp)).
- **OpenAI** ✅ — Code Interpreter is one of the longest-running code-mode tools.
- **E2B** ✅ — productizes sandbox-as-a-service for code-mode agents.
- **Modal** ✅ — sandboxes for code-mode agents.
- **Cognition (Devin)** ⚠ — heavy code-mode use.
- **Cursor agent mode, Replit Agent, Continue** ⚠ — code-mode by default.
- **HuggingFace smolagents** ✅ — open-source library promoting code-mode.
- **Anthropic Computer Use** ⚠ — programmatic computer control is a form of code mode.
- **Manus, Open Devin, Open Hands** ⚠ — open-source code-mode agents.

## Further reading

- [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — Anthropic Nov 2025 (canonical)
- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024 (broader framing)
- [E2B documentation](https://e2b.dev/docs) — sandbox-as-a-service
- [HuggingFace smolagents](https://github.com/huggingface/smolagents) — code-mode library
- [OpenAI Code Interpreter](https://platform.openai.com/docs/assistants/tools/code-interpreter)
- [Modal sandbox docs](https://modal.com/docs/guide/sandboxes)
- [Pyodide docs](https://pyodide.org/en/stable/)
- [On the road to general LLM agents](https://huggingface.co/blog/smolagents) — HuggingFace smolagents intro

---

*Diagram source: [`../diagrams/src/code-mode.d2`](../diagrams/src/code-mode.d2)*
