# Structured Outputs / Function Calling

**Aliases:** JSON mode, schema-constrained outputs, function calling, tool calling, structured generation, constrained decoding
**Category:** Foundations
**Sources:**
[OpenAI — Function calling (2023)](https://platform.openai.com/docs/guides/function-calling) ·
[OpenAI — Introducing Structured Outputs (Aug 2024)](https://openai.com/index/introducing-structured-outputs-in-the-api/) ·
[Anthropic — Tool use docs (2024-2026)](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use) ·
[Willard & Louf — Efficient Guided Generation (Outlines, 2023)](https://arxiv.org/abs/2307.09702) ·
[XGrammar paper (Microsoft, 2024)](https://arxiv.org/abs/2411.15100)

---

## Problem

> [!TIP]
> **ELI5.** Your program needs to *use* the model's output — pass it to another function, store it in a database, send it as an API call. But LLMs naturally produce free text, and free text breaks parsers. **Structured outputs** are the family of techniques that make the model produce *valid JSON (or some other schema) guaranteed*. The simplest version is "just ask nicely in the prompt." The strongest version constrains the model's *sampling step* so it can't physically emit invalid output. In between are JSON mode and function calling. By 2026, the strongest version (constrained decoding) is available on every major hosted model and most open-source inference servers — so there's basically no excuse for production code to be parsing free-text LLM responses anymore.

LLMs that emit unstructured prose are nice for chatbots and bad for software. Every real agentic application needs the model to produce *parseable* output: a function call with named arguments, a JSON object with specific fields, an action plan in a typed format. Without structure, you can't connect the LLM to a tool, a database, or another LLM in a workflow.

Through 2022-2024 this was *the* main practical-engineering problem with LLMs. Models could be coaxed into "JSON-shaped" output by prompting, but they'd hallucinate fields, drift formats, wrap their JSON in markdown fences, add chatty preambles, or just produce malformed syntax on bad days. Production teams spent significant effort on parser tolerance, retry-with-fix loops, and schema validation.

The 2024-2025 shift: hosted platforms (OpenAI, Anthropic, Google, Mistral, Cohere) and OSS libraries (Outlines, Guidance, llguidance, XGrammar, llama.cpp grammars) made **schema-guaranteed structured output** a first-class feature. By 2026, you can hand the model a JSON Schema (or Pydantic / Zod type) and get back output that's *provably* valid against that schema. This unlocked a lot of agentic-pattern reliability — tool use, multi-agent message passing, and workflow orchestration all rely on it.

## How it works

> [!TIP]
> **ELI5.** Three tiers from weak to strong. **Tier 1**: ask in the prompt ("respond as JSON with these fields"). Cheap, works on any model, fails sometimes. **Tier 2**: use the platform's JSON mode or function calling — the model is more careful, the response is more often valid. **Tier 3**: constrained decoding — the model literally *cannot* emit invalid output because the sampling step rejects any token that would break the schema. By 2026, Tier 3 is widely available; use it whenever you can.

![Structured outputs — three tiers](../diagrams/svg/structured-outputs.svg)

### Tier 1: Prompt-only structuring

The naive approach: in your prompt, describe the desired format.

```
You are a JSON-producing assistant. For each input, respond with valid JSON 
of the form: {"intent": string, "entities": list[string], "confidence": float}.
Do not include any other text.
```

Pros:
- Works on any model, including pure open-source non-tuned ones.
- Zero infrastructure required.
- Fastest to prototype.

Cons:
- Failure rate of 5-15% on most non-tuned models. Failures include: malformed JSON syntax, missing required keys, type errors (string instead of int), trailing commentary ("Sure, here's the JSON: ..."), markdown fences (` ```json ... ``` `).
- Requires defensive parsing — wrap in try/except, regex-strip code fences, retry on failure.
- Quality drops further with smaller models.

Tier 1 is for prototyping. Don't ship it.

### Tier 2: JSON mode and function calling

Hosted platforms started shipping "JSON mode" (OpenAI, late 2023) and "function calling" (OpenAI, mid-2023; Anthropic 2024) — features that *encourage* the model to produce JSON and validate the output at the server before returning.

**JSON mode** roughly: "the response will be valid JSON syntactically, but the contents may not match your intended schema." Server-side validation rejects non-JSON; the model is also fine-tuned to produce JSON when this mode is on.

**Function calling / tool use** roughly: "the model emits a `function_name` and `arguments` object matching a tool schema." The platform validates the structure (function exists, arguments is JSON). The contents still might be wrong, but the *shape* is guaranteed.

This is what every major SDK (OpenAI, Anthropic, Google, Mistral, Cohere, Vercel AI SDK, LangChain, LlamaIndex) defaults to. Reliable enough for most production.

Cons:
- Still not *guaranteed* schema-valid at the content level. A field that's supposed to be an enum might come back with an invalid value; a required nested field might be missing.
- Tied to specific platforms; less control over the constraints.
- Some quality cost in some cases (older models do worse with strict modes).

### Tier 3: Constrained decoding (the 2024-2026 default)

The strongest version: constrain the *sampling step itself* so that the model can't physically emit a token that would violate the schema. At each generation step, the inference engine zeros out the probability of any token that would lead to invalid output, and re-samples from the allowed set.

This is **provably schema-valid by construction** — there's no "the model didn't follow the schema" failure mode. There's only "the schema was satisfied but the values were wrong."

Implementations:
- **OpenAI Structured Outputs** (August 2024). Pass a JSON Schema or Pydantic model; the model is guaranteed to produce schema-valid output. Available on GPT-4o, o-series, and successors.
- **Anthropic tool use with strict mode**. Tool argument schemas are strictly enforced.
- **Outlines** (Willard & Louf, 2023). OSS library; works with any open-weight model via grammar-constrained sampling. Supports regex, JSON Schema, EBNF grammars.
- **Guidance** (Microsoft, 2023). Similar to Outlines, slightly different DSL.
- **XGrammar** (Microsoft, 2024). High-performance constrained decoding kernels — minimal speed overhead.
- **llguidance** (Microsoft, 2024). Newer high-performance library.
- **llama.cpp grammars**. GBNF grammar files for constraining llama.cpp inference.
- **vLLM, SGLang, TensorRT-LLM** all support some form of guided/constrained generation.

By late 2025, constrained decoding is the production default for any team that cares about reliability.

### Schema vs validation vs content

Three distinct properties to distinguish:

1. **Syntactic validity.** The output is parseable JSON. Tier 1 fails here sometimes; Tier 2 and 3 essentially never fail here.
2. **Schema validity.** The output matches your schema (required fields present, types correct, enums respected). Tier 3 guarantees this; Tier 2 *usually* delivers; Tier 1 fails frequently.
3. **Content correctness.** The values in the output are actually right. *No tier guarantees this.* Constrained decoding ensures the model produces *some* enum value, but not the *correct* enum value.

The bug a lot of teams hit: assuming Tier 3 fixes correctness. It doesn't. It fixes parseability and schema-conformance. Correctness still requires good prompts, evals, and human review.

### Common schema patterns

In practice, these schema patterns recur across products:

- **Single classification.** `{"label": <enum>, "confidence": float}`. The most common production use.
- **Entity extraction.** `{"entities": [{"text": str, "type": <enum>, "start": int, "end": int}]}`.
- **Tool call.** `{"function": <enum of tool names>, "arguments": <tool-specific schema>}`. The function-calling case.
- **Plan.** `{"steps": [{"action": str, "tool": str, "args": ...}]}`.
- **Multi-field decision.** `{"decision": <enum>, "rationale": str, "next_action": str}`.
- **Workflow routing.** `{"route_to": <enum of branches>, "reason": str}`. The classifier in [routing patterns](../agt/workflows-vs-agents.md).

For each, define a Pydantic model (Python) or Zod schema (TypeScript), pass to the platform, get typed objects back.

### Composing structured outputs with reasoning

A subtle 2024-2026 question: should the model *reason* before producing the structured output? Two patterns:

**Reasoning-then-structure.** Prompt: "First, think step by step. Then produce the JSON answer." The model writes reasoning in prose, then JSON at the end. The structured-output engine extracts the JSON. Reasoning is preserved (for debugging) but not part of the schema.

**Reasoning inside structure.** Schema includes a `reasoning` field: `{"reasoning": str, "final_answer": str}`. Both produced together.

The first pattern preserves more reasoning quality (model isn't constrained while reasoning). The second is more compact and easier to parse. Both common.

With reasoning models (o1, Claude extended thinking): the model reasons *internally* (not visible) and produces a structured *final* output. This is often the cleanest setup.

### Costs and gotchas

- **Constrained decoding has marginal speed cost.** Modern implementations (XGrammar, llguidance) are within ~5% of unconstrained inference. Older implementations were much slower.
- **Schemas with many enums / complex nesting** can constrain the model into low-quality answers. The model picks the least-bad valid option. Mitigation: simplify schemas, add an "other" / "unknown" enum value.
- **Deeply nested schemas** can blow up token usage and confuse the model. Keep schemas flat where possible.
- **Optional fields and unions** add complexity. Default to required + null where reasonable.
- **Long schemas** eat your context budget. Don't include unnecessary fields.
- **Always validate after**. Even Tier 3 — defense in depth. Wrap in Pydantic / Zod / Joi parse; retry on failure with the error.

### The retry-with-fix pattern

Even with Tier 3 you may want a retry on failure (for the rare validation issue or content-level rejection):

```
1. Generate structured output.
2. Validate against full schema + business rules.
3. On failure: feed the error back to the model:
   "Your previous output failed validation: <error>. Try again."
4. Cap retries (2-3) to avoid infinite loops.
```

Most production stacks include this loop. Libraries like Instructor (Python) and Vercel AI SDK with retries handle it built-in.

### Versioning schemas

Schemas evolve. Best practices:
- Version your schemas (don't silently change them).
- Add new optional fields rather than changing existing ones.
- When you must break, run old and new schemas in parallel temporarily.
- Test against fixtures of old data when schemas change.

### When NOT to use structured outputs

- **Free-form generation where structure would harm quality.** Long-form writing, creative tasks, conversational replies.
- **Cases where you actually want prose followed by structure** — use reasoning-then-structure pattern.
- **Highly variable schemas you can't predict.** Open-ended extraction may need post-hoc parsing.
- **Pure chat UI** where the response is shown directly to a user.

## Variants & related patterns

- [**Augmented LLM**](augmented-llm.md) — structured outputs enable the "tools" augmentation.
- [**ReAct**](react.md) — typically uses structured tool calls.
- [**Single-agent with tools**](../agt/single-agent-with-tools.md) — relies on structured tool call schemas.
- [**Multi-agent orchestration**](../agt/multi-agent-orchestration.md) — agents pass structured messages to each other.
- [**Workflows vs agents**](../agt/workflows-vs-agents.md) — workflow routing classifiers use structured outputs.
- [**Maker-checker**](../agt/maker-checker.md) — the checker reads a structured verdict.
- **Pydantic, Zod, JSON Schema** — the schema-definition languages.
- **Instructor, Marvin, LangChain output parsers** — wrapping libraries.
- **CodeAct / Program-aided** — alternative where the "structured output" is generated code that's executed.

## When NOT to use (recap)

See above. Default to using it; carve out exceptions.

## Implementations

| Tool / platform | Tier | Notes |
|---|---|---|
| **OpenAI Structured Outputs** | Tier 3 | Native, JSON Schema or Pydantic |
| **OpenAI function calling** | Tier 2-3 | strict: true enables Tier 3 |
| **Anthropic tool use** | Tier 2-3 | Strict mode for Claude 3.5+ |
| **Google Gemini function calling** | Tier 2-3 | controlled generation feature |
| **Mistral / Cohere structured outputs** | Tier 2-3 | platform-specific |
| **Outlines (OSS)** | Tier 3 | works with any HF model |
| **Guidance (OSS, Microsoft)** | Tier 3 | DSL for constrained generation |
| **XGrammar, llguidance** | Tier 3 | High-perf kernels under hood |
| **llama.cpp grammars (GBNF)** | Tier 3 | Local inference |
| **vLLM, SGLang, TensorRT-LLM** | Tier 3 | Server-side guided generation |
| **Instructor (Python), Vercel AI SDK** | Wrapper | Typed schema → response, retry-with-fix |
| **DSPy** | Higher-level | Programmatic with typed signatures |

## Companies / products built on structured outputs

- **OpenAI** ✅ — function calling, structured outputs as first-class features.
- **Anthropic** ✅ — tool use with strict schemas.
- **Google Gemini** ✅ — controlled generation, function calling.
- **Hosted RAG products** ✅ — Perplexity, You.com, Phind — all use structured intermediate states.
- **Coding agents** ✅ — Cursor, Aider, Cline, Devin — all use structured tool calls.
- **Stripe AI features** ✅ — structured outputs for financial actions.
- **Notion AI, Linear AI, Asana AI** ✅ — extract structured data from prose.
- **Outlines, dottxt, Anthropic-internal tooling** ✅ — drive open-weight model adoption of Tier 3.

## Further reading

- [Introducing Structured Outputs in the API](https://openai.com/index/introducing-structured-outputs-in-the-api/) — OpenAI, Aug 2024
- [Anthropic Tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use) — Anthropic docs
- [Efficient Guided Generation for LLMs](https://arxiv.org/abs/2307.09702) — Willard & Louf (Outlines paper)
- [XGrammar: Flexible and Efficient Structured Generation](https://arxiv.org/abs/2411.15100)
- [Instructor: Structured outputs powered by LLMs](https://python.useinstructor.com/)
- [Outlines documentation](https://outlines-dev.github.io/outlines/)
- [Guidance documentation](https://github.com/guidance-ai/guidance)
- [Marvin AI (Prefect)](https://www.askmarvin.ai/) — typed-output framework

---

*Diagram source: [`../diagrams/src/structured-outputs.d2`](../diagrams/src/structured-outputs.d2)*
