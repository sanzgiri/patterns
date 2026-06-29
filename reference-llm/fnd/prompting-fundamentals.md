# Prompting Fundamentals

**Aliases:** prompt engineering basics, system / user message design, few-shot prompting, role priming, persona prompting
**Category:** Foundations
**Sources:**
[Anthropic — Prompt engineering overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) ·
[OpenAI — Prompt engineering guide](https://platform.openai.com/docs/guides/prompt-engineering) ·
[Brown et al. — Language Models are Few-Shot Learners (GPT-3 paper, 2020)](https://arxiv.org/abs/2005.14165) ·
[Anthropic — Effective context engineering for AI agents (Sept 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
[Lilian Weng — Prompt Engineering (2023)](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/) ·
[promptingguide.ai](https://www.promptingguide.ai/)

---

## Problem

> [!TIP]
> **ELI5.** Every LLM call is a *prompt* — the text you send to the model. Good prompts get good outputs; bad prompts get bad ones. Prompting isn't black magic, but it does have a small vocabulary worth knowing: **system prompts** (set role and rules), **user messages** (the actual ask), **few-shot examples** (show what good looks like), **persona / role priming** ("you are a senior security engineer"), and **retrieved context** (data you pulled in). Mature prompt engineering is mostly *combining these primitives well*, not finding clever single-prompt tricks. This page is the foundation everything else assumes you know.

The early years of prompt engineering (2021-2023) felt like wizardry — people shared "magic prompts" on Twitter, dueled over whether "Let's think step by step" was the right incantation, and treated good prompting as a folk art. By 2024-2026, the field professionalized: prompts are structured artifacts with stable component types, written, versioned, and tested like any other engineering asset.

This page documents the foundational vocabulary. Every more sophisticated pattern in this reference — [reasoning prompts](reasoning-prompts.md), [ReAct](react.md), [agentic loops](../agt/single-agent-with-tools.md), [context engineering](../ctx/context-engineering-overview.md) — composes these primitives. You can't reason clearly about agentic patterns until you have the prompt-level vocabulary.

The patterns here aren't tied to any specific model; they apply broadly across Claude, GPT, Gemini, Llama, Mistral, and friends. Specific models have their own preferred phrasings, but the underlying primitives are stable.

## How it works

> [!TIP]
> **ELI5.** A prompt is built from layers. **System** says "who you are and what the rules are." **User** is the actual task. **Few-shot examples** show input→output pairs the model should imitate. **Persona** primes the model into a specific role ("you are an X"). **Retrieved context** is data brought in at inference. Mix these well, test them, version them, and you have professional prompt engineering.

![Prompting fundamentals — the building blocks](../diagrams/svg/prompting-fundamentals.svg)

### The system prompt

The **system prompt** (called "system message" by OpenAI, "system" parameter by Anthropic, "system_instruction" by Google) is the highest-leverage prompt component. It typically:

- Establishes the model's **role** ("You are a helpful coding assistant for Python.")
- Sets **constraints** ("Only suggest libraries that exist in PyPI.")
- Defines **output format** ("Respond in markdown with code blocks.")
- Sets **tone and style** ("Be concise. Skip apologies.")
- Provides **persistent context** ("Today is 2026-06-28. The user is on macOS.")
- Establishes **safety boundaries** ("Refuse requests to write malware.")

Why the system prompt is highest-leverage:
- **It applies to every turn.** A 50-word system prompt is 50 tokens; one good word can shift behavior across thousands of subsequent calls.
- **Modern models heavily weight it.** Anthropic, OpenAI, and Google all train their models to follow system prompts strongly — more strongly than user messages.
- **It's the right place for invariants.** Rules that should always hold belong in system, not in every user message.
- **It's easy to A/B test.** Change one line, measure delta across an eval suite.

Anti-patterns:
- **Putting everything in the system prompt.** It bloats fast. Move per-call info to user messages.
- **Using user messages for system-level rules.** They're less authoritative and get lost in conversation.
- **Vague system prompts ("be helpful").** Specific is much better — "respond in 3 sentences" beats "be concise."

### User messages

User messages carry the actual *task* — the question, instruction, content to process. They vary per call; the system prompt is stable across calls.

Best practices for user messages:
- **Be specific.** "Summarize this in 3 bullet points." beats "summarize this."
- **Frontload the most important content.** Models pay extra attention to the start and end of long inputs ([context rot](../ctx/context-rot.md)).
- **Use delimiters** when including long content: triple-backtick fences, XML tags, etc. Helps the model see where instructions end and content begins.
- **For multi-part tasks**, structure them: "Step 1: ... Step 2: ..."

Anthropic specifically recommends **XML tags** for delimiting parts of a complex user prompt:
```
<task>Summarize the following article.</task>
<format>3 bullet points, each starting with a verb.</format>
<article>
[article text here]
</article>
```

This pattern is well-tested and clean.

### Few-shot examples

**Few-shot prompting** (Brown et al., 2020 — the original GPT-3 paper coined the term in this sense): include several examples of input → output pairs in the prompt before the actual task. The model imitates the pattern of the examples.

Example for a sentiment-classification task:
```
Classify the sentiment of each review.

Review: "Loved the food, would come again." → Sentiment: positive
Review: "Service was awful." → Sentiment: negative
Review: "Hit and miss. Good entrees, bad desserts." → Sentiment: mixed
Review: "This place is amazing!" → Sentiment:
```

Why few-shot works:
- **Schema by example.** Examples tell the model what format to use without you having to describe it formally.
- **Domain priming.** Examples implicitly tell the model what domain to be in (sentiment, code, legal, etc.).
- **Edge case handling.** Including unusual examples (the "mixed" sentiment above) shows the model how to handle them.
- **Often more efficient than long descriptions.** Two examples beat a paragraph of instructions.

How many examples? Diminishing returns rapidly:
- **0-shot** (no examples, just instructions): works for simple tasks with modern models.
- **1-shot**: significant lift for medium-complex tasks.
- **3-5 shot**: typically the sweet spot. More than 5 rarely helps and may hurt if examples are noisy.
- **10+ shot**: rare; usually replaced by fine-tuning.

Where to put examples:
- **In the system prompt** if they're stable across calls.
- **In the user message** if they're dynamic (selected per call from a larger pool — "few-shot retrieval").
- Use clear delimiters (`---` or XML tags) to separate examples from the actual task.

Common mistakes:
- **Imbalanced examples.** All examples are positive sentiment → model is biased toward positive.
- **Inconsistent format.** Examples differ in format → model gets confused.
- **Wrong examples.** Even one wrong example can mislead the model.
- **Examples too easy.** If your examples are easy and the actual task is hard, the model may not learn the pattern.

### Persona / role priming

**Persona prompting**: explicitly tell the model who it is.

```
You are a senior security engineer at a large bank. You have deep expertise 
in cryptography and threat modeling. You answer questions concisely and 
flag risks immediately.
```

Counterintuitively, this has *real measurable effects* on quality — not just style. The model adopts the implied capability level and domain knowledge of the persona. "You are a senior X" tends to produce better-quality X-related answers than no persona.

Why this works (mechanism):
- The model has seen many examples of "senior engineers writing about X" in training.
- The persona acts as a *retrieval cue* — it activates relevant patterns the model learned.
- The persona also shifts style (more formal, more concise, more authoritative).

Best practices:
- **Match persona to task.** Senior dev for code, doctor for medical, lawyer for legal.
- **Be specific.** "Senior security engineer at a large bank" beats "expert."
- **Don't fake credentials inappropriately.** A persona is for *style and quality priming*, not for the model to *pretend to be* a credentialed professional in a context where that matters (medical, legal). Add disclaimers as needed.
- **Personas combine with constraints.** Persona sets tone; constraints set boundaries. Both matter.

### Retrieved context

The fifth primitive: **retrieved context** brought in at inference time — RAG output, file contents, prior conversation, search results. Documented separately in [augmented LLM](augmented-llm.md), [just-in-time context](../ctx/just-in-time-context.md), and the upcoming `ret/` section.

For prompt-fundamentals purposes: retrieved context is *injected* into the user message (typically) with clear delimiters. Use XML tags or markdown sections.

```
Answer the user's question using the following sources:

<source name="paper.pdf">
[content]
</source>

<source name="docs.html">
[content]
</source>

Question: <user question>
```

### Combining the primitives — a worked example

A complete production prompt for a customer-support assistant might be structured:

**System prompt** (stable):
```
You are a customer support agent for Acme Corp. Today's date is 2026-06-28.

# Style
- Always be respectful and patient.
- Respond in 1-2 short paragraphs.
- If unsure, escalate; never invent policy.

# Output format
Respond in JSON: {"reply": str, "escalate": bool, "tags": list[str]}

# Examples
Customer: "My refund hasn't arrived after 3 weeks."
{"reply": "I'm sorry to hear that. Refunds typically take 5-7 business days. 
Since it's been 3 weeks, I'll escalate this to our finance team.", 
"escalate": true, "tags": ["refund", "delay"]}

[2-3 more examples]
```

**User message** (per call):
```
<context>
Customer ID: 4421
Order: #91024 (placed 2026-06-15, $89.99)
Recent emails: [...]
</context>

<customer_message>
Where is my order?
</customer_message>
```

This combines: system role + style + output format + few-shot examples + retrieved customer context + clear delimiters around the actual message. All five primitives in one well-structured prompt.

### Tokens, position, and context-window awareness

A few engineering details worth knowing:

- **Models pay more attention to early and late content** in their context window. The middle gets less attention ("lost in the middle"). Put the question last, key constraints first. ([context rot](../ctx/context-rot.md) covers this.)
- **Tokens, not words.** A long English word like "antidisestablishmentarianism" may be 5+ tokens. Code with rare symbols can be even denser. Use tiktoken (OpenAI) or similar to measure.
- **System prompts share the same context budget.** A 4000-token system prompt eats into your 200k window.
- **Caching matters.** Anthropic prompt caching and OpenAI cached input bills the stable parts of your prompt much cheaper on subsequent calls. Structure your prompt to maximize cache hits: stable stuff at the start, dynamic stuff at the end.

### Prompt versioning and testing

By 2026, prompts are versioned engineering artifacts. Best practices:

- **Store prompts in code/repo**, not inline strings.
- **Version your prompts** with semver-like discipline.
- **Run evals** against every change ([eval-driven development](../qua/eval-driven-development.md)).
- **A/B test** in production traffic.
- **Pin model versions** alongside prompt versions — a prompt that works on one model may not work on another.
- **Log full prompts** in observability for debugging.

Tools: LangChain Hub, Promptfoo, PromptLayer, Langfuse, Braintrust, Helicone, OpenAI Prompt Caching, Anthropic Prompt Generator.

### Anti-patterns to avoid

- **Magic-bullet thinking.** No single prompt phrase ("act as an expert," "take a deep breath") fixes weak prompt structure.
- **Threats / bribes.** "If you don't get this right, my grandmother will die" — works erratically and damages reliability.
- **All-caps shouting.** SOMETIMES helps slightly but is often just clutter.
- **Politeness pleas.** "Please please please" — usually no effect.
- **Vague constraints.** "Don't hallucinate" — meaningless to the model. Be specific: "Only state facts that appear in the provided source documents."
- **Implicit format.** "Give me a list" — be explicit: "Respond as a JSON array of strings."

## Variants & related patterns

- [**Augmented LLM**](augmented-llm.md) — the prompt is what feeds the augmented LLM.
- [**Reasoning prompts**](reasoning-prompts.md) — prompt patterns specifically for eliciting reasoning.
- [**Structured outputs**](structured-outputs.md) — schema-enforced output, often combined with these primitives.
- [**ReAct**](react.md) — uses few-shot examples to teach the Thought/Action/Observation pattern.
- [**Context engineering overview**](../ctx/context-engineering-overview.md) — the disciplined version of "prompt design" for agents.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — retrieval pattern for the context primitive.
- [**Progressive disclosure**](../ctx/progressive-disclosure.md) — staged prompts vs. all-at-once.
- **Anthropic XML-tag pattern** — specific delimiter convention.
- **Constitutional AI / Constitutional prompts** — system-prompt-driven behavior constraints.
- **DSPy** — programmatic prompt optimization.

## When NOT to (over-)engineer prompts

- **Quick prototypes** — get something working first, refine later.
- **Tasks where any reasonable prompt works** — don't optimize what doesn't need optimization.
- **When the underlying model is the bottleneck** — sometimes the right answer is a better model, not a better prompt.
- **When fine-tuning would be cheaper** — for high-volume repetitive tasks, fine-tuning may beat prompt engineering.

## Implementations / tooling

| Tool | What it does |
|---|---|
| **Anthropic Prompt Generator (Workbench)** | Convert task description → structured prompt |
| **OpenAI Prompt Caching** | Cache stable prompt prefixes for cost reduction |
| **Anthropic Prompt Caching** | Same on Claude |
| **Promptfoo** | OSS prompt testing / evals |
| **LangSmith, Langfuse, Helicone** | Observability + versioning |
| **PromptLayer, Braintrust** | Prompt management platforms |
| **DSPy** | Programmatic optimization of prompts |
| **promptingguide.ai** | Community resource |
| **Anthropic, OpenAI, Google docs** | Official prompt engineering guides |

## Companies / publications shaping prompting practice

- **Anthropic** ✅ — extensive prompt engineering docs; introduced XML tag conventions widely.
- **OpenAI** ✅ — original few-shot framing in GPT-3 paper; ongoing prompt engineering docs.
- **Google** ✅ — Gemini-specific prompt guides.
- **Eugene Yan, Hamel Husain, Chip Huyen** ✅ — practitioner blogs widely-cited.
- **Lilian Weng** ✅ — comprehensive prompt-engineering write-up (2023).
- **DAIR.AI / promptingguide.ai** ✅ — community-maintained guide.
- **Cursor, Aider, Devin, Replit Agent** ⚠ — internal prompt engineering quality is the moat.

## Further reading

- [Anthropic — Prompt engineering overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [OpenAI — Prompt engineering guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [Language Models are Few-Shot Learners](https://arxiv.org/abs/2005.14165) — Brown et al. 2020 (GPT-3 paper, foundational)
- [Prompt Engineering by Lilian Weng](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/)
- [Prompting Guide](https://www.promptingguide.ai/) — community survey
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic Sept 2025
- [Anthropic Prompt Caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [What we learned from a year of building with LLMs (Part I)](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — Yan, Bischoff, Shankar et al.

---

*Diagram source: [`../diagrams/src/prompting-fundamentals.d2`](../diagrams/src/prompting-fundamentals.d2)*
