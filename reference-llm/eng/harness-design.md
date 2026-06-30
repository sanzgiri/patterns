# Harness Design (The Agent OS)

**Aliases:** agent harness, agent runtime, agent OS, LLM application chassis, agent framework internals
**Category:** Production Engineering
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[Dex Horthy — 12-Factor Agents (2024)](https://github.com/humanlayer/12-factor-agents) ·
[Vercel AI SDK documentation](https://sdk.vercel.ai/docs) ·
[Mastra documentation](https://mastra.ai/docs) ·
[OpenAI Agents SDK (2025)](https://platform.openai.com/docs/) ·
production practice: every serious LLM application has one

---

## Problem

> [!TIP]
> **ELI5.** A raw LLM API call is just `prompt → completion`. That's not enough for production. Real LLM apps need: context assembly (system prompt + memory + RAG + tools), input validation, model selection (brain vs hands), retries on transient errors, timeouts, cost caps, structured output parsing, schema validation, tool execution (sandboxed!), output guardrails, post-hoc verification, telemetry for everything. **The harness** is the code that wraps the LLM call and handles all of that. It's the "agent OS" — the platform layer between the bare model API and your application logic. Most production failures come from harness bugs (silent context truncation, missing retries, no timeouts, no observability) rather than from the LLM itself. Frameworks like Vercel AI SDK, Mastra, LangGraph give you a harness; custom builds give you ownership of it.

The 12-Factor Agents discussion ([12-factor-agents](12-factor-agents.md)) frames it precisely: the LLM is the engine; the harness is the chassis. Engine-only is a quarter-mile drag race; chassis-plus-engine is a car you can drive every day.

The pattern is invisible when it works and catastrophic when it doesn't. A common production tale: a system goes down for 2 hours because the LLM API returned a malformed structured output and the harness didn't have schema validation; the malformed output went into the database; downstream systems broke. The fault was the harness, not the model.

By 2026 the landscape of "harness frameworks" has consolidated around a few families. But the core responsibilities are universal regardless of framework choice.

## How it works

> [!TIP]
> **ELI5.** Eight distinct concerns, each a layer of the harness. Run them in order around every LLM call. Some are cheap (validation, parsing); some involve LLM calls themselves (routing, post-hoc verification). The harness owns the orchestration; the LLM owns the reasoning. Together they're an agent.

![Harness design](../diagrams/svg/harness-design.svg)

### The 8 standard harness layers

**1. Context assembly.** Gather everything that goes into the prompt:
- System prompt (versioned, possibly per-route)
- Memory injection (working / episodic / semantic — see [memory architectures](../mem/memory-architectures.md))
- Retrieved chunks (RAG — see [basic-rag](../ret/basic-rag.md))
- Tool descriptions
- Few-shot examples
- Conversation history (or its compaction)
- The user's current message
- Any structured context (user metadata, session info)

Assembly order matters for [prompt caching](../mem/prompt-caching.md): static first, dynamic last.

**2. Input validation + safety filters.** Before paying for an LLM call:
- Token budget check (fits in context window?)
- PII / sensitive content scrubbing
- Prompt injection screening
- Rate limit per user
- Auth check (does this user have access to these tools?)

**3. Router / model selection.** Pick the right model for the job:
- [Brain vs hands](brain-vs-hands.md) dispatch
- Vendor selection (OpenAI / Anthropic / Google / OSS)
- Model-version pinning vs latest
- Specialized model for specific tasks (code model, embedding model, vision model)

**4. LLM call with retries + timeout + cost cap.** The actual API invocation, but defensively:
- Timeout per call (typically 30-120s)
- Retry budget on transient errors (HTTP 5xx, 429, network)
- Exponential backoff
- Cost cap per call (and per session)
- Streaming vs non-streaming based on use case
- Vendor failover (if Anthropic is down, fall back to OpenAI)

**5. Structured output parsing + schema check.** Validate the model's output:
- JSON parse (constrained decoding helps but doesn't guarantee)
- Schema validation (Pydantic, Zod, JSON Schema)
- Fail fast on schema violation; retry with feedback in prompt
- Fall back gracefully if all retries fail

**6. Tool execution.** When the model says "call this tool":
- Sandbox the execution ([browser as sandbox](../sec/browser-as-sandbox.md), [containment](../sec/containment-blast-radius.md))
- Apply egress rules ([egress allowlisting](../sec/egress-allowlisting.md))
- Capture results, errors, side effects
- Feed results back to next LLM turn

**7. Output guardrails + post-hoc verification.** Before returning to user:
- Refusal / safety classifier on output
- Hallucination check (often a cheap LLM judge)
- Citation verification (do cited sources actually exist?)
- Business rule checks (no profanity, no PII leakage)
- Length / format final checks

**8. Telemetry / trace logging.** Log everything:
- Full prompt + response (with PII redaction)
- Model, version, parameters
- Token counts, cost
- Latency (TTFT and total)
- Cache hit/miss
- Tool calls made
- Each layer's pass/fail
- User feedback (when available)

Each layer's instrumentation feeds [LLM observability](llm-observability.md).

### Why this many layers

Each layer prevents a specific class of production failure:

- **No context assembly discipline** → silent truncation, model misses critical info.
- **No input validation** → cost / rate-limit attacks, prompt injection.
- **No router** → expensive model on trivial calls (or vice versa).
- **No retry / timeout** → flaky calls hang the system.
- **No schema validation** → malformed outputs poison downstream code.
- **No tool sandboxing** → agent has unrestricted access to your system.
- **No output guardrails** → model says something terrible to a user.
- **No telemetry** → you can't debug or improve.

You can skip layers in a prototype. You can't skip them in production at scale.

### What good harness frameworks give you

Modern harness frameworks (Vercel AI SDK, Mastra, LangGraph, OpenAI Agents SDK, Pydantic AI) implement most of these for you:

- **Context assembly**: structured prompt-building primitives.
- **Streaming**: server-sent events, websocket, etc.
- **Tool definitions**: typed function signatures.
- **Structured outputs**: schema-first with auto-validation.
- **Retries / timeouts**: configurable defaults.
- **Tracing**: built-in observability hooks (often LangSmith / Braintrust integration).
- **Multi-provider**: same code works across Anthropic / OpenAI / Google.

What they typically DON'T give you (you build yourself):
- **Domain-specific safety filters.**
- **Business-rule guardrails.**
- **Tool sandboxing details** (the framework gives you the hook; the sandbox is yours).
- **Per-tier cost caps.**
- **Specific eval logic.**
- **A/B testing infrastructure.**

### The "build vs framework" decision

**Use a framework when:**
- You're starting fresh and don't have strong opinions yet.
- The framework's idioms match your use case.
- You value time-to-market over perfect control.
- You're using one of the well-supported model providers.

**Build your own when:**
- You have unusual requirements (special compliance, custom routing).
- You hit framework limits and find yourself fighting it.
- You need very tight cost control.
- You're at scale where framework overhead matters.

A common pattern: start with a framework, gradually migrate critical paths to custom code as you learn what you need.

### Concrete framework choices (2026)

| Framework | Stack | Best for |
|---|---|---|
| **Vercel AI SDK** | TypeScript | Web apps, Next.js, streaming UI |
| **Mastra** | TypeScript | Workflow-heavy TS apps |
| **LangGraph** | Python/TS | Stateful graphs, complex flows |
| **OpenAI Agents SDK** | Python | OpenAI-first, sub-agents |
| **Pydantic AI** | Python | Type-safe; FastAPI-friendly |
| **DSPy** | Python | Programmatic optimization |
| **LangChain (v0.3+)** | Python/TS | Broad ecosystem |
| **Mastra, Inngest, Trigger.dev** | TS | Long-running workflows |
| **Custom** | Any | Full control, more work |

### Common harness anti-patterns

- **No context-assembly discipline.** The prompt is built ad-hoc per call; no central control.
- **Hidden token truncation.** When context overflows, the harness silently drops content. Symptom: random quality regressions.
- **No retries on transient errors.** Single 5xx fails the user request.
- **No timeout.** Stuck model calls hang the harness.
- **Same retry budget for all errors.** Don't retry a 400 (your bug), do retry a 503.
- **No cost cap.** One bad day = bankrupting bill.
- **No schema validation.** Malformed JSON propagates.
- **Tool execution without sandbox.** Agent runs arbitrary code on your machine.
- **No telemetry.** You can't see what's happening.
- **Logging full prompts to plain text.** PII leakage; compliance violation.
- **Mixing logging with prompt construction.** Debug spam pollutes prompts.

### Production engineering specifics

- **Idempotency keys** on outbound tool calls (especially writes to external systems).
- **Cancellation propagation**: when the user navigates away, cancel pending LLM calls.
- **Deduplication**: same prompt twice in quick succession → return cached result.
- **Per-route SLOs**: different routes have different latency budgets.
- **Graceful degradation**: when LLM is slow/down, return a stub or fall back to non-LLM logic.
- **Feature flags for prompts/models**: ship prompt or model changes via flags, roll back instantly.
- **Cost telemetry per user**: detect runaway usage; cap or alert.
- **Replay capability**: log enough to re-run any session offline for debugging.

### What "good harness" looks like in practice

You can drop into any session log and find:
- The exact prompt sent (every token).
- The exact response.
- The model + parameters.
- The cost + latency.
- Any tool calls made and their results.
- The eval score (if one was applied).
- The user's feedback (if any).
- Any error or guardrail flag.

If you can find all of that for any past session, your harness is production-grade. If you can't, harness is the work to do.

## Variants & related patterns

- [**12-Factor Agents**](12-factor-agents.md) — informs the harness design.
- [**Brain vs hands**](brain-vs-hands.md) — model routing at the harness layer.
- [**Rainbow deployments**](rainbow-deployments.md) — deployment infrastructure for harness changes.
- [**LLM observability**](llm-observability.md) — what the harness instruments.
- [**Augmented LLM**](../fnd/augmented-llm.md) — the atom the harness wraps.
- [**Structured outputs**](../fnd/structured-outputs.md) — used at layer 5.
- [**Prompt caching**](../mem/prompt-caching.md) — context assembly hooks for caching.
- [**Memory architectures**](../mem/memory-architectures.md) — memory loaded at layer 1.
- [**Maker-checker**](../agt/maker-checker.md) — verification often lives at layer 7.
- [**Containment & blast radius**](../sec/containment-blast-radius.md) — layer 6 boundary.

## When NOT to invest heavily

- **Prototypes** where you're still learning the problem.
- **One-off scripts.**
- **Demo / hackathon code.**
- **Internal-only tools** with one trusted user.

## Implementations

| Framework | Harness completeness |
|---|---|
| **Vercel AI SDK** | High; web/streaming-focused |
| **Mastra** | High; workflow-focused |
| **LangGraph** | Medium-high; flexible |
| **OpenAI Agents SDK** | Medium-high; OpenAI-first |
| **Pydantic AI** | Medium-high; type-first |
| **DSPy** | Medium; programmatic style |
| **LangChain v0.3+** | Medium; broad |
| **AutoGen** | Medium; multi-agent focus |
| **CrewAI** | Medium; multi-agent focus |
| **Custom** | Variable |

## Companies / products with strong harness engineering

- **Anthropic** ✅ — internal harness referenced in [Building effective agents](https://www.anthropic.com/research/building-effective-agents).
- **OpenAI** ✅ — Assistants → Agents SDK evolution shows harness maturation.
- **GitHub Copilot Workspace** ⚠ — production harness for code agents.
- **Cursor, Continue, Cody** ⚠ — well-engineered harnesses for code editing.
- **Cognition Devin** ⚠ — sophisticated long-running harness.
- **HumanLayer** ✅ — productizes harness layer for enterprises.
- **Glean, Vectara, Pinecone Assistant** ✅ — enterprise RAG harnesses.
- **Inflection Pi, Character.AI** ⚠ — character + memory harnesses.
- **Vercel AI SDK / Mastra / LangChain (LangGraph) / Pydantic AI** ⚠ — productize the harness pattern.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024
- [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) — Horthy
- [Vercel AI SDK documentation](https://sdk.vercel.ai/docs)
- [Mastra documentation](https://mastra.ai/docs)
- [LangGraph documentation](https://langchain-ai.github.io/langgraph/)
- [OpenAI Agents SDK docs](https://platform.openai.com/docs/agents)
- [Pydantic AI](https://ai.pydantic.dev/)
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [What we learned from a year of building with LLMs](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/)

---

*Diagram source: [`../diagrams/src/harness-design.d2`](../diagrams/src/harness-design.d2)*
