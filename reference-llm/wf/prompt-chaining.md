# Prompt Chaining

**Aliases:** sequential LLM workflow, pipeline pattern, multi-step LLM decomposition, gated chains
**Category:** Workflows
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[LangChain documentation on chains (2023-2026)](https://python.langchain.com/docs/concepts/) ·
[OpenAI Agents SDK / Assistants pipelines (2024-2026)](https://platform.openai.com/docs/) ·
production practice: Eugene Yan, Hamel Husain, Chip Huyen on LLM pipelines

---

## Problem

> [!TIP]
> **ELI5.** One big LLM prompt that does everything ("read this article, summarize it, translate the summary to French, then format as a tweet") often fails on at least one step — and you can't tell which. **Prompt chaining** is the simplest fix: do it as a sequence of smaller LLM calls, one per step. Each call has a clearly-defined input and output. You can validate between steps, retry on failure, and use different models for different steps. It's the *first workflow pattern you should reach for* when a task has identifiable sub-steps — cheaper, simpler, and more reliable than one giant prompt.

Anthropic's [Building effective agents](https://www.anthropic.com/research/building-effective-agents) (Dec 2024) post is the canonical reference for the five workflow patterns. Of those five, **prompt chaining is the one to try first** for any non-trivial task. The argument: many tasks that *seem* like they need an agent are actually sequential workflows where each step is well-defined. Workflows like this are dramatically cheaper, more predictable, and easier to debug than a tool-using agent.

The pattern is decades-old in software engineering (pipes, Unix shell, ETL, function composition). What's novel in the LLM context is that each "function" in the chain is a probabilistic LLM call. That probabilistic step needs the same defensive engineering — input validation, error handling, retries — that you'd apply to any flaky external call.

## How it works

> [!TIP]
> **ELI5.** Take your task, split it into ordered sub-tasks. Each sub-task gets its own LLM call. Pipe the output of one into the input of the next. Add a *programmatic gate* between any two calls to validate the intermediate result (right format, sensible content) and short-circuit on failure. You now have a deterministic outer structure with probabilistic inner steps — best of both worlds.

![Prompt chaining](../diagrams/svg/prompt-chaining.svg)

### The mechanics

A prompt chain is a fixed sequence of LLM calls, defined by code (not the model):

```
input  →  LLM call 1  →  [gate?]  →  LLM call 2  →  [gate?]  →  LLM call 3  →  output
```

Each LLM call:
- Has a **specific, narrow purpose** ("generate outline," "expand to draft," "polish style").
- Has a **defined input contract** — what it expects.
- Has a **defined output contract** — what it returns (often [structured output](../fnd/structured-outputs.md)).
- Can use a **different model** if appropriate (cheap model for easy steps, strong model for hard).
- Can have **different prompts, examples, system messages** — tuned for that step.

Each gate (optional, between calls):
- **Validates** the previous call's output (schema check, content rules, business logic).
- **Decides** whether to proceed, retry, fail, or route elsewhere.
- Is **deterministic** code, not an LLM call (usually).
- Is the *primary lever* on overall chain reliability.

### Example — content production chain

A common production chain for AI-assisted content:

1. **Call 1** — "Generate a 5-bullet outline for a blog post on this topic."
   - Output: structured JSON with `headline` and `bullets[]`.
   - Gate: validate that there are exactly 5 bullets, each ≤ 12 words.
2. **Call 2** — "Expand each bullet into a paragraph."
   - Input: the outline JSON.
   - Output: structured JSON with `headline`, `paragraphs[]`.
   - Gate: validate paragraphs aren't truncated, hit a target length.
3. **Call 3** — "Polish the draft for tone and clarity."
   - Input: assembled draft.
   - Output: final markdown.
   - Gate: no more than 5% word count change.

Versus the single-prompt version ("write me a blog post on X"), the chain:
- **Surfaces failures.** If the outline is bad, you see it before the draft is generated.
- **Lets you tune each step.** Different prompts, different models per step.
- **Caches well.** Steps 1 and 2 results cache; re-running step 3 doesn't regenerate everything.
- **Is debuggable.** Each call's input/output is logged separately.

### Why chaining is cheaper and more reliable than one big prompt

**Quality.** Each call is asked to do one thing. LLMs do "one thing" much better than "many things at once." Failure modes compound multiplicatively in a single big prompt; in a chain, each failure is isolated.

**Cost.** Smaller, more-focused prompts means fewer tokens per call. Even though there are more calls, total tokens are often *lower* than one mega-prompt that has to repeat all context.

**Model mixing.** Easy steps go to fast/cheap models (GPT-4o-mini, Haiku, Gemini Flash). Hard steps go to strong models. A chain typical 2026 saves 5-10× on inference cost vs. all-strong-model.

**Caching.** Each step's output can be cached deterministically. Re-running with the same step-1 input skips re-generating it.

**Debuggability.** Logs per step. When something breaks, you see exactly which step failed.

**Iterability.** Tweak one step's prompt without disturbing the others.

### Choosing between chaining and a single prompt

Use a chain when:
- The task has 2+ clearly identifiable sub-steps.
- Sub-steps have different ideal models (cheap → strong, or vice versa).
- You want to validate / gate between steps.
- Caching intermediate results saves significant cost.
- You need to be able to retry partial failures.

Use a single prompt when:
- The task is genuinely atomic (one short answer).
- The sub-steps are tightly coupled (output of one can't be cleanly described without the next).
- Latency budget can't tolerate the extra round-trips.
- The chain doesn't actually improve quality (always measure with evals).

### Gates — the key reliability mechanism

The gate between LLM calls is what makes chaining robust. Without gates, errors propagate; with gates, they're caught early.

Gate categories:
- **Schema gates.** Did the output match the expected schema? Constrained-decoding output ([structured outputs](../fnd/structured-outputs.md)) is essentially a guaranteed schema gate.
- **Content gates.** Did the output satisfy business rules? (Length, tone, format, presence of required entities.)
- **Quality gates.** A separate LLM call ([model grader](../qua/grader-taxonomy.md)) scores the output.
- **Safety gates.** Refusal-style checks — does the output contain prohibited content?
- **Length gates.** Output not too long / not too short.

When a gate fails:
- **Retry** with feedback in the prompt ("Your previous output had X problem. Try again.").
- **Re-prompt** with adjusted parameters (lower temperature, different examples).
- **Route to fallback** — e.g., a different model or a different chain.
- **Fail fast** — return an error to the caller; don't waste compute on broken intermediate state.

The retry-with-fix loop is dominant in production. Most chains include 1-3 retries per step with the validation error fed back.

### Common variants

- **Static chain.** Fixed sequence of N calls; no branching, no looping. Simplest, most common.
- **Conditional chain.** A step's output decides which next step runs. Borderline with [routing](routing.md).
- **Chain with refinement loop.** A step that calls itself with self-critique until a quality threshold is met. Borderline with [evaluator-optimizer](evaluator-optimizer.md).
- **Map chain.** Apply the same chain to each item in a list. Often parallelized; see [parallelization](parallelization.md).
- **Hybrid: chain → agent.** Static chain for setup; tool-using agent for an exploratory final step. Common for research-style tasks.

### Engineering details that matter

- **Per-step timeouts.** A stuck LLM call hangs the whole chain; cap each.
- **Per-step retries.** Independent retry budgets per step.
- **Idempotency keys.** When chains run in production with retries, idempotency keys prevent duplicate side effects.
- **Trace logging.** Each step's prompt, output, model, latency, tokens — all logged. Essential for debugging.
- **Versioned prompts.** Step prompts should be versioned independently. Eval each version.
- **Cost telemetry per step.** Know which step is the cost driver; optimize that one.
- **Partial-success handling.** If step 3 of 5 fails, return what you have or roll back? Depends on the use case.

### Anti-patterns

- **No gates.** Errors propagate silently; debugging is awful.
- **Same model for every step.** Likely overpaying on easy steps or under-resourcing hard steps.
- **Unbounded loops.** Refinement chains without iteration caps spin forever on adversarial inputs.
- **Steps too small.** A chain with 12 micro-steps adds latency without much quality gain.
- **Steps too large.** A chain with 2 mega-steps is barely a chain — the original "big prompt" problem in two pieces.
- **Skipping evals.** Each step needs its own evals; whole-chain evals are insufficient.

### How this composes with agentic patterns

Chains and agents aren't mutually exclusive:
- A **chain can be embedded in an agent** — one step of an agent's plan can run a chain.
- An **agent can be embedded in a chain** — one step of a chain can be an agent that does open-ended work.
- The right tool depends on the step: deterministic structure → chain; open-ended exploration → agent.

In practice, mature LLM products are *mostly chains with selective agent use*, not the other way around. Workflows are the boring backbone; agents are the special-case capability.

## Variants & related patterns

- [**Routing**](routing.md) — chain step that picks downstream branches.
- [**Parallelization**](parallelization.md) — running independent steps concurrently.
- [**Orchestrator-workers**](orchestrator-workers.md) — dynamic version where an LLM plans the chain.
- [**Evaluator-optimizer**](evaluator-optimizer.md) — generator + critic as a refining loop.
- [**Workflows vs agents**](../agt/workflows-vs-agents.md) — the meta-distinction this fits into.
- [**Augmented LLM**](../fnd/augmented-llm.md) — each chain step is an augmented LLM call.
- [**Structured outputs**](../fnd/structured-outputs.md) — typical inter-step contract.
- [**Maker-checker**](../agt/maker-checker.md) — formal version of gate-based validation.

## When NOT to use

- **Atomic tasks** without clean sub-step structure.
- **Latency-critical** paths where multiple round-trips are too expensive.
- **Truly open-ended tasks** where the next step depends on rich exploration — use an agent.
- **Cases where one model can do it well in one call** — measure with evals before adding complexity.

## Implementations

| Framework | Chain support |
|---|---|
| **LangChain LCEL** | Pipe-style composition |
| **LangGraph** | Graph-based chains (with branching, conditionals) |
| **Vercel AI SDK** | Sequential `generateText` / `generateObject` calls |
| **OpenAI Agents SDK (2025)** | Native pipeline primitives |
| **DSPy** | Programmatic chain compilation and optimization |
| **Haystack** | Pipeline-first framework |
| **LlamaIndex Query Pipelines** | DAG-style chains |
| **Mastra** | TS-first workflow primitives |
| **Pydantic AI** | Type-safe chains |
| **Plain async/await** | Often the right answer — no framework needed |

## Companies / products using chains

Essentially every production LLM product. Representative:

- **Anthropic** ✅ — recommends chaining as the first workflow ([source](https://www.anthropic.com/research/building-effective-agents)).
- **OpenAI** ✅ — Assistants and Agents SDK are chain-friendly.
- **GitHub Copilot Workspace** ⚠ — plan-then-execute is essentially a chain.
- **Notion AI, Linear AI, Stripe AI** ⚠ — structured chains for extract → enrich → respond.
- **Perplexity, Phind, You.com** ⚠ — query → retrieve → rerank → answer chains.
- **Most enterprise RAG deployments** ✅ — chain shapes the canonical RAG pipeline.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024 (canonical workflow patterns)
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [What we learned from a year of building with LLMs](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — Yan, Bischoff, Shankar et al.
- [LangChain LCEL guide](https://python.langchain.com/docs/concepts/lcel/)
- [DSPy: Compiling Declarative Language Model Calls](https://arxiv.org/abs/2310.03714) — Khattab et al. (programmatic chain optimization)

---

*Diagram source: [`../diagrams/src/prompt-chaining.d2`](../diagrams/src/prompt-chaining.d2)*
