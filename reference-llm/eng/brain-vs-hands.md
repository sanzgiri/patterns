# Brain vs Hands (Reasoning + Executor Model Split)

**Aliases:** planner-executor model split, two-tier model architecture, supervisor + worker models, big-model + small-model decomposition
**Category:** Production Engineering
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) (advocates small-model + big-model split) ·
[LangChain — Supervisor pattern (2024-2025)](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/) ·
[Cursor — model routing blog](https://www.cursor.com/blog) (Cursor "fast" vs "auto" vs "max" tiers) ·
production practice: Devin, Replit Agent, Claude Code, GitHub Copilot Workspace

---

## Problem

> [!TIP]
> **ELI5.** Production agents do many different things — high-stakes reasoning ("what's the right approach?") AND mundane formatting ("convert this to JSON"). If you use one strong model for everything, you pay reasoning-model prices for trivial work and suffer reasoning-model latency on the hot path. If you use one cheap model for everything, hard reasoning is bad. **Brain vs hands** is the solution: use TWO models. The **brain** is an expensive reasoning model (Claude Opus, GPT-5, Gemini Ultra, o3) that plans, decides, and reflects. The **hands** is a cheap fast model (Claude Haiku, GPT-4o-mini, Gemini Flash) that handles formatting, parsing, classifying, calling tools, summarizing results. The harness routes each step to brain or hands based on what it needs. Common split: ~20% of calls go to the brain, ~80% to hands — but ~80% of cost is brain. Result: 5-10× cost reduction with quality parity or improvement on most production agents.

The pattern is suggested in passing by Anthropic's [Building effective agents](https://www.anthropic.com/research/building-effective-agents) ("use the cheapest model that meets your bar") and made explicit by LangChain's [supervisor architecture](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/), Cursor's tier system, and countless production deployments.

The economic case is straightforward by 2026:
- Frontier reasoning models: Claude Opus ~$15/Mtok in, GPT-5 ~$10/Mtok, o3 ~$60/Mtok
- Workhorse models: Claude Haiku 4.5 ~$1/Mtok, GPT-4o-mini ~$0.15/Mtok, Gemini Flash ~$0.075/Mtok

That's a 10-100× cost spread. Tasks where the cheap model is "good enough" should never see the expensive model.

But the case is also about *quality and latency*: cheap fast models are *better* at deterministic format-shaped tasks (they don't overthink), and avoid putting your hot path behind a 5-30s reasoning-model call.

## How it works

> [!TIP]
> **ELI5.** Look at every LLM call in your agent and ask: "does this need actual reasoning, or is it pattern-matching / formatting / classification?" Route the reasoning ones to a big model; everything else to a small model. The split is usually obvious once you look. Wrap both behind your harness; the caller asks for "brain" or "hands" and the harness picks.

![Brain vs hands](../diagrams/svg/brain-vs-hands.svg)

### What goes to the brain

- **Planning** — what's the next step? what's the overall approach?
- **Reasoning** — multi-step inference, trade-off analysis
- **Reflection** — did this work? what went wrong?
- **Open-ended generation** — writing, code authoring, design
- **Hard math / coding** — the actual problem-solving
- **Decision-making with consequences** — which tool? which branch?

### What goes to the hands

- **Tool call formatting** — convert "search for X" into the right JSON
- **Structured extraction** — parse this output into fields
- **Classification / routing** — is this a refund, support, or sales question?
- **Summarization** — condense this 5k-token tool result into 500 tokens
- **Translation / rephrasing** — say this in a friendlier tone
- **Boilerplate generation** — formatted error messages, status updates
- **Schema validation prep** — clean output to match expected schema

### Concrete example: a coding agent

Walking through what a Cursor / Claude Code session looks like with brain-vs-hands:

1. **User**: "Refactor the auth module to use JWT instead of session cookies."
2. **Brain**: Reads the request, the AGENTS.md, the directory structure. Plans: "Find all session-cookie usages, group by file, refactor each, update tests, verify."
3. **Hands**: For each file, "extract the imports and exports" — classify which need changing.
4. **Brain**: For each file group, generates the actual refactor diff.
5. **Hands**: Formats each diff into the tool call shape; classifies file types.
6. **Brain**: After tests run, reflects on failures and decides next steps.
7. **Hands**: Summarizes test output ("12 tests pass, 2 fail — auth_login_test.py: assertion at line 45; user_session_test.py: timeout").
8. **Brain**: Uses the summary to plan fixes.

Of the LLM calls in this session, the brain handled ~5 (planning, generation, reflection). The hands handled ~30 (formatting, extraction, summarization). The hands' 30 calls cost less than 1 brain call.

### Why the split also helps quality

Counter-intuitively, the cheap model is often *better* at the hands tasks:

- **Schema adherence**: small models trained heavily on JSON / structured outputs hit the schema cleanly. Big models occasionally "explain" instead of formatting.
- **Speed for routing**: a 200ms classifier call doesn't slow the user-facing path; a 5s reasoning call does.
- **Specificity**: a model with a narrow task prompt (the "hands" call) often does it better than the same model with a long mixed prompt.
- **Stability**: small-model output variance on routine tasks is *lower* than big-model variance.

You're not just saving money — you're often *improving* the deterministic parts of your pipeline.

### Implementation patterns

**Pattern 1: Per-call routing in harness.**
```python
result = await llm.call(
    task="format_tool_call",
    tier="hands",   # routes to small model
    input=user_msg
)
plan = await llm.call(
    task="plan_next_step",
    tier="brain",   # routes to big model
    input=state
)
```

The harness owns the model registry; the caller declares intent ("brain" / "hands"); the harness picks the actual model. Easy to change later.

**Pattern 2: Supervisor + workers** ([LangGraph supervisor](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/)).
- Supervisor (brain) orchestrates.
- Worker agents (hands) execute specific tasks.
- Supervisor never does the actual work; workers never plan.

**Pattern 3: Tier-based UX** (Cursor, Claude Code).
- User picks "auto," "fast," "max" mode.
- Mode determines model tier.
- User trades cost/latency for quality.

**Pattern 4: Adaptive routing.**
- Classifier (hands) determines task difficulty.
- Easy → hands all the way. Hard → escalate to brain.
- Self-tuning based on outcomes.

### Cost analysis

For a typical production agent with mixed calls:

| Setup | Avg cost per session | Quality |
|---|---|---|
| Brain-only (frontier model everywhere) | $0.50 | High |
| Hands-only (cheap model everywhere) | $0.04 | Mediocre on planning |
| **Brain + hands split** | $0.10 | High (brain on reasoning, hands on formatting) |

Brain-only: paying premium for trivial work.
Hands-only: cheap but planning suffers.
Split: 5× cheaper than brain-only, quality matches.

This is consistently the cheapest path to quality parity in production.

### When ONE model is the right call

- **Tiny apps** where the harness complexity isn't worth it.
- **Specialized domains** where one fine-tuned model dominates.
- **Reasoning-heavy tasks** where even "format the output" benefits from the reasoning model.
- **Strict consistency requirements** where mixing models introduces variance you can't accept.
- **Vendor lock-in or contract constraints.**

### Engineering details

- **Vendor diversification.** Brain and hands don't need to be the same vendor. Common: Claude Opus brain + Gemini Flash hands.
- **Per-tier metrics.** Track cost, latency, quality per tier independently.
- **Routing telemetry.** Track which tasks go to which tier; surfaces misrouting.
- **Per-tier prompt tuning.** Hands prompts should be terse and structured; brain prompts can be richer.
- **Per-tier caching.** Both tiers benefit from [prompt caching](../mem/prompt-caching.md).
- **Fallback chain.** If hands fails, escalate to brain. Audit when this fires.
- **Cost cap per tier.** Separate budgets so a brain-leak doesn't bankrupt you.

### Anti-patterns

- **Brain on the hot path.** Frontier models add seconds; a chat UI feels slow.
- **Hands for reasoning.** Cheap models miss multi-step inference; quality suffers.
- **Mixing roles in one prompt.** "Plan AND format" defeats the split.
- **No routing telemetry.** You can't tune what you don't measure.
- **Same prompt for both tiers.** Defeats the per-tier optimization.
- **Hard-coded model names.** Switching tiers later is painful.
- **No cost cap per tier.** A bug routes everything to brain; cost explodes.

### How brain-vs-hands relates to other patterns

- **vs [orchestrator-workers](../wf/orchestrator-workers.md)**: orchestrator-workers is a *structural* pattern (one boss, many workers); brain-vs-hands is a *model-tier* pattern (one expensive model, one cheap). Orthogonal — often combined.
- **vs [routing](../wf/routing.md)**: routing dispatches by intent; brain-vs-hands dispatches by task difficulty.
- **vs [reasoning models](../model/reasoning-models.md)**: reasoning models are *one* kind of brain.
- **vs single-tier deployment**: most newcomers start single-tier; brain-vs-hands is the first cost optimization.

### Trajectory through 2026

The brain-vs-hands distinction is becoming explicit in vendor APIs:

- **Anthropic**: Claude Haiku / Sonnet / Opus family, plus extended thinking on / off — explicit tiers.
- **OpenAI**: GPT-4o-mini / GPT-4o / GPT-5 / o-series — explicit tiers.
- **Google**: Gemini Flash / Pro / Ultra + Thinking — explicit tiers.
- **Routing-as-a-service**: OpenRouter, Portkey, LiteLLM, Helicone now offer auto-routing.

The pattern will continue to be central — if anything, MORE so as reasoning models get more expensive but more capable, widening the brain/hands cost ratio.

## Variants & related patterns

- [**12-Factor Agents**](12-factor-agents.md) — brain-vs-hands is one architecture compatible with the principles.
- [**Harness design**](harness-design.md) — the harness is where the routing logic lives.
- [**Routing**](../wf/routing.md) — closely related; routing by intent vs by tier.
- [**Orchestrator-workers**](../wf/orchestrator-workers.md) — structural cousin.
- [**Reasoning models**](../model/reasoning-models.md) — what often goes in the brain slot.
- [**Single agent with tools**](../agt/single-agent-with-tools.md) — common form factor.
- [**Coding agents**](../agt/coding-agents.md) — heavy users of the pattern.
- [**Prompt caching**](../mem/prompt-caching.md) — multiplies the cost benefit.

## When NOT to use

- **Tiny apps** where the engineering overhead doesn't pay back.
- **Strict-consistency deployments** where mixing models is unacceptable.
- **One-off scripts** with no cost sensitivity.
- **Specialized fine-tuned models** where one model serves all needs.

## Implementations

| Framework | Brain-vs-hands support |
|---|---|
| **LangGraph supervisor** | First-class pattern |
| **OpenAI Agents SDK** | Handoffs support tier-based dispatch |
| **Vercel AI SDK** | Plain code routing between `streamText` calls |
| **Mastra** | Native model tier support |
| **DSPy** | Programmatic tier selection |
| **OpenRouter, Portkey, LiteLLM** | Hosted routing infrastructure |
| **Helicone, Braintrust** | Routing + observability |
| **Custom harness** | Trivially supported with provider abstraction |

## Companies / products using brain-vs-hands

- **Cursor** ✅ — explicit `auto`/`fast`/`max` tiers with model routing.
- **Claude Code** ⚠ — uses Haiku and Sonnet internally for different roles.
- **Cognition Devin** ⚠ — multi-tier internally.
- **GitHub Copilot** ⚠ — completion (cheap fast) vs Chat (mid) vs Workspace (strong).
- **Replit Agent** ⚠ — multi-tier model use.
- **OpenAI ChatGPT (auto-model routing)** ⚠ — routes user queries to reasoning vs non-reasoning models.
- **Anthropic Claude.ai** ⚠ — Haiku/Sonnet/Opus selection.
- **Customer support AI** (Intercom Fin, Zendesk AI) ✅ — cheap classifier + strong responder.
- **Most cost-optimized RAG systems** ⚠ — cheap embedder + cheap reranker + strong generator.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024
- [LangGraph supervisor tutorial](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/)
- [Cursor blog on model routing](https://www.cursor.com/blog)
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [OpenRouter routing docs](https://openrouter.ai/docs)
- [Helicone — model routing](https://docs.helicone.ai/features/router)
- [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) — Horthy

---

*Diagram source: [`../diagrams/src/brain-vs-hands.d2`](../diagrams/src/brain-vs-hands.d2)*
