# Routing

**Aliases:** classifier routing, query routing, intent routing, dispatcher pattern, branch selection
**Category:** Workflows
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[OpenAI — Routing patterns in Agents SDK (2025)](https://platform.openai.com/docs/) ·
[Eugene Yan — Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) ·
production practice: customer-support routing, multi-tier model deployments

---

## Problem

> [!TIP]
> **ELI5.** Not all inputs deserve the same treatment. Customer asks "how do I reset my password?" → cheap-FAQ-bot answer. Customer asks "I lost $5,000, refund pending for 6 weeks" → escalate to a tool-using agent with refund authority. Customer asks "your API returns a 502 on /v2/charges" → strong reasoning model with technical-docs RAG. **Routing** is the workflow pattern that does this dispatch: a classifier LLM (or rules engine) looks at the input and picks the right downstream branch. Each branch is tuned for its category. Result: better quality where it matters, lower cost where it doesn't.

Routing is Anthropic's second canonical workflow pattern (after [prompt chaining](prompt-chaining.md)). The key insight: when you have multiple categories of input that benefit from *different prompts, different tools, different models*, routing lets each category get its specialized treatment while keeping the calling code unified.

Two problems routing solves that a single all-purpose prompt can't:

1. **Quality dilution.** A prompt that handles all cases tends to be average at each. A prompt specialized to one case is better at that case.
2. **Cost mismatch.** Routing simple inputs to cheap models and hard inputs to strong models is the easiest 5-10× cost optimization in most LLM products.

By 2026, routing is essentially universal — virtually every production LLM product routes at multiple levels (model tier, agent skill, tool selection, content category).

## How it works

> [!TIP]
> **ELI5.** First LLM call classifies the input into a category (or set of categories). A simple `if`/`else` (or table lookup) picks the right downstream prompt / model / agent. Each branch is independently optimized for its category. The caller sees one entry point; the implementation has many specialized backends.

![Routing — classifier picks the right specialized branch](../diagrams/svg/routing.svg)

### The mechanics

1. **Receive input.** User query, document, ticket, or other unit of work.
2. **Run a classifier.** Usually an LLM call constrained via [structured output](../fnd/structured-outputs.md) to an enum: `{"category": <one of: A, B, C, ...>}`. Optionally with confidence or rationale.
3. **Dispatch on category.** Plain code: `if category == "A": handle_A(input)`.
4. **Each branch handles its specialized case.** Different prompt, possibly different model, possibly different tools or RAG sources.
5. **Optionally merge.** If the branches all return the same response type, the caller sees a uniform response.

### Examples by routing axis

**Routing by intent (customer-support pattern):**
```
"How do I reset my password?"       → faq-bot (cheap model + FAQ docs RAG)
"My refund hasn't arrived"          → refund-agent (tool-using agent with refund API)
"Your API returns 502"              → tech-support-agent (strong model + tech docs RAG)
"I want to cancel my account"       → retention-flow (specialized prompt + escalation)
```

**Routing by complexity (cost optimization):**
```
"What is the capital of France?"    → mini model, no tools, no thinking
"Plan my weekend in Paris."         → full model, web search tool
"Solve this multi-step proof."      → reasoning model with extended thinking
```

**Routing by capability (agent-skill dispatch):**
```
User asks → router LLM
            "is this about: code? math? scheduling? writing? web search?"
                 ↓
            specialized agent / skill for that capability
```

**Routing by safety / risk:**
```
Normal queries          → standard model
Potentially-sensitive   → model with stronger refusal training + safety filters
Known-risky topics      → block / escalate to human
```

### Why routing works

The argument is essentially the specialization argument from microservices (see system-design `arch/microservices`), restated for LLMs:

- **A general-purpose endpoint serves many use cases moderately well**; a specialized endpoint serves one use case extremely well.
- **A general-purpose prompt** has to handle many edge cases, with token budget split across them. Each specialized prompt can be optimized for its one case.
- **A general-purpose model** is the expensive frontier model that works for everything. Each specialized branch can choose the cheapest model that meets its bar.
- **Failure isolation**: a bug in one branch's prompt doesn't affect other branches.

### Designing the classifier

The classifier is the load-bearing piece. Key design decisions:

**Number of categories.** Too few (2-3): not enough specialization. Too many (20+): hard to be accurate, hard to maintain. Sweet spot is usually 4-8.

**Mutually exclusive vs multi-label.** Most production routers use mutually exclusive (one category per input). Multi-label is harder to handle downstream — each branch may need to handle being called alongside others.

**Use [structured outputs](../fnd/structured-outputs.md).** Constrain the classifier to emit *only* a valid category. Don't parse free text.

**Include a fallback category.** "other" / "unknown" routes to a safe default branch. Real-world inputs always include cases your categories don't cover.

**Include confidence (optional).** `{"category": "refund", "confidence": 0.9}`. Below a threshold → route to fallback or escalate.

**Examples in the classifier prompt.** Few-shot examples per category dramatically improve classifier accuracy.

**Cheap classifier model.** Routers run on every input. Use a small/fast model (Haiku, Flash, GPT-4o-mini) unless the classification is genuinely hard.

### When a non-LLM classifier suffices

For some routing tasks, an LLM classifier is overkill. Use the cheaper option when you can:

- **Keyword/regex.** "Does input contain 'refund'? → refund branch." Cheap and deterministic.
- **Embedding similarity.** Embed input + each category description; pick max-similarity. Fast.
- **Rules / decision tree.** Hand-coded for stable categories.
- **Traditional classifier.** Fine-tuned BERT or similar — production-grade, very fast.

An LLM classifier is best for nuanced category discrimination where keyword/embedding aren't accurate enough. Many production routers are *hybrid* — cheap classifier first, fall through to LLM for ambiguous cases.

### Cost analysis of routing

Routing has a small fixed cost (the classifier LLM call) and a potentially large variable saving (cheap downstream branches for common cases).

```
no routing:    every query → strong model ($0.05 each)
with routing:  classifier ($0.001) + branch (avg $0.015) → ~70% saving
```

The math works whenever:
- Cheap branches handle a meaningful share of traffic (≥30%).
- The classifier doesn't error often (≥95% accuracy).
- The classifier itself is cheap relative to the savings.

For products at scale (millions of queries/day), routing is one of the highest-ROI optimizations.

### Composing routing with other patterns

Routing combines with the other workflow patterns:

- **Routing + chaining.** Each branch is itself a chain. Common: customer-support router → chain per category.
- **Routing + parallelization.** Multi-label routing fans out to multiple branches concurrently.
- **Routing + agent.** Router picks "is this a structured task or open-ended?" → workflow vs. agent.
- **Hierarchical routing.** Router → sub-router → branch. Common for large category spaces.
- **Routing in orchestrator-workers.** Orchestrator routes per sub-task to specialized workers.

### Anti-patterns

- **Routing every query through the strongest model.** Defeats the cost benefit. Use a cheap classifier.
- **Too many categories.** Hard to maintain consistency; classifier accuracy drops.
- **Brittle classifier prompts.** Without few-shot examples, classifiers drift. Eval the classifier independently.
- **No fallback category.** Real inputs include weird ones. Have a safe default.
- **Routing without observability.** When the wrong branch handles a query, debugging is hard without per-route metrics.
- **Hard-coded routes that should be dynamic.** If your categories change quarterly, a hard-coded `if` chain is technical debt.
- **Classifying when keyword would do.** LLM classifier cost adds up at scale. Use the cheaper method when accuracy allows.

### Production engineering

- **Eval the router separately.** Treat the classifier as its own product with its own accuracy metric. Confusion matrix per category. ([Capability vs regression evals](../qua/capability-vs-regression-evals.md)).
- **Per-route metrics.** Latency, cost, error rate, user satisfaction per branch. Surfaces routing-quality problems.
- **Telemetry on misroutes.** When user feedback says "this was the wrong category," log it for classifier improvement.
- **Versioned categories.** Adding/removing categories is a breaking change. Treat the category enum as a versioned schema.
- **A/B test new branches.** When adding a new specialized branch, A/B against the previous routing.
- **Drift detection.** Category distribution shifts over time. Alert when traffic patterns change significantly.

### Routing in the agentic era

A subtle 2025-2026 development: routing is sometimes done *implicitly* by tool-using agents. The agent has many tools; it implicitly "routes" by picking which tool to call. This is routing in a different shape:

- **Explicit routing**: code decides the branch based on classifier output.
- **Implicit routing**: an agent decides which tool/sub-agent to call.

Explicit routing is more controllable and observable; implicit routing is more flexible. Mature systems use both, often hierarchically (explicit router at the top, agent for open-ended branches).

## Variants & related patterns

- [**Prompt chaining**](prompt-chaining.md) — each branch may itself be a chain.
- [**Parallelization**](parallelization.md) — multi-label routing → fan-out.
- [**Orchestrator-workers**](orchestrator-workers.md) — dynamic version where the orchestrator routes per-task.
- [**Evaluator-optimizer**](evaluator-optimizer.md) — generator/critic loops are usually downstream of routing.
- [**Workflows vs agents**](../agt/workflows-vs-agents.md) — the meta-pattern.
- [**Multi-agent orchestration**](../agt/multi-agent-orchestration.md) — sub-agent dispatch is routing.
- **API gateway pattern** (system-design `comm/api-gateway`) — the analogous system-design pattern.
- **Strangler fig** (system-design `arch/strangler-fig`) — routing as migration tool.

## When NOT to use

- **Tiny scale.** If you're handling 100 queries/day, the routing complexity isn't worth it.
- **Truly homogeneous inputs** that all benefit from the same treatment.
- **When all your "categories" are actually one task with parameters** — solve with prompt parameterization, not routing.
- **Where routing latency is unacceptable.** The classifier round-trip adds latency.

## Implementations

| Framework | Routing support |
|---|---|
| **LangGraph** | Conditional edges + custom routers; first-class graph pattern |
| **LangChain RouterChain** | Native router primitive |
| **OpenAI Agents SDK (2025)** | Handoffs as a routing primitive |
| **Vercel AI SDK** | `generateObject` for classifier + plain code dispatch |
| **DSPy** | Programmatic router compilation |
| **LlamaIndex** | RouterQueryEngine; pydantic-based selectors |
| **Mastra** | Workflow with routing primitives |
| **Promptfoo / LangSmith / Braintrust** | Router-specific eval support |
| **OpenRouter, Portkey, LiteLLM** | Model-routing-as-infrastructure |
| **Plain code** | `if classifier_output == "X": handle_X()` — often perfectly fine |

## Companies / products using routing

- **Anthropic** ✅ — recommends in [Building effective agents](https://www.anthropic.com/research/building-effective-agents).
- **OpenAI ChatGPT** ⚠ — internal model routing (ChatGPT picks reasoning vs non-reasoning model per query).
- **GitHub Copilot** ⚠ — routes per task (code completion vs chat vs Workspace).
- **Customer-support products** ✅ — Intercom, Zendesk, Salesforce all route customer queries.
- **Perplexity** ⚠ — routes queries to web search vs Pro Search vs reasoning model.
- **Cursor, Aider, Continue** ⚠ — route between code-edit, code-chat, and agent modes.
- **OpenRouter, Portkey, Helicone, LiteLLM** ✅ — productize cross-vendor routing.
- **Most cost-optimized LLM products** ⚠ — cheap-model-first routing is the easiest cost lever.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [LangGraph routing tutorials](https://langchain-ai.github.io/langgraph/)
- [What we learned from a year of building with LLMs (Part II)](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-ii/) — covers routing patterns
- [DSPy programmatic routing](https://dspy-docs.vercel.app/)
- [OpenRouter docs on model routing](https://openrouter.ai/docs)

---

*Diagram source: [`../diagrams/src/routing.d2`](../diagrams/src/routing.d2)*
