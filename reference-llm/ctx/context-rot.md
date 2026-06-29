# Context Rot

**Aliases:** context dilution, attention dilution, lost-in-the-middle (a specific instance)
**Category:** Context Engineering
**Sources:**
[Anthropic — Effective context engineering for AI agents (Sept 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) ·
[Liu et al. 2023 — Lost in the Middle](https://arxiv.org/abs/2307.03172) ·
Needle-in-a-haystack benchmarking lineage (Greg Kamradt, Anthropic, et al. 2023-2025)

---

## Problem

> [!TIP]
> **ELI5.** You hand someone a 500-page book and ask them to find the one sentence that mentions your dog's name. They'll do okay if the sentence is on page 1 or page 500 — but probably miss it if it's buried in chapter 14, surrounded by 200 other pages of vaguely related content. LLMs have the same problem with their context window: stuff in the middle, surrounded by lots of other tokens, becomes harder for them to *actually use*.

A common assumption through 2023-2024: bigger context windows = better. The intuition was that if a model can fit a million tokens, you can just pour your entire corpus into the prompt and ask it questions — no need for RAG, no need for compaction, no need for clever curation.

The reality, documented across two years of benchmarks (Liu et al.'s "Lost in the Middle" in 2023, then RULER, NoLiMa, BABILong, Anthropic's internal needle-in-haystack work in 2024-2025): **as context grows, a model's ability to use information from that context degrades**. Not catastrophically — there's no hard cliff — but along a gradient. By the time you reach 500K-1M tokens, even frontier models miss "needles" they would trivially find at 8K tokens.

Anthropic gave the phenomenon a name in their September 2025 context-engineering post: **context rot**. The implication: even with 1M+ token windows, context must be treated as a precious, finite resource. The cure isn't bigger windows — it's better curation.

## How it works (the mechanism)

> [!TIP]
> **ELI5.** Three things make long context hard. (1) Every word the model reads has to "pay attention to" every other word — that's a multiplication, not an addition, so doubling the context quadruples the work. (2) The model was trained on mostly shorter text, so it's just *less practiced* at handling huge contexts. (3) The math the model uses to track word positions has to be stretched to handle positions it wasn't trained on, and the stretching adds noise.

Three architectural realities of transformer LLMs combine to produce context rot:

![Attention budget — the n² problem](../diagrams/svg/attention-budget.svg)

**1. Quadratic attention cost.** A transformer attends from every token to every other token. With `n` tokens that's `n²` pairwise relationships. Doubling the context quadruples the cost — and quadruples the demand on the model's learned attention patterns. The model still produces an answer, but the *quality of attention* on any single token in a long window is lower than the quality of attention on the same token in a short window. Anthropic calls this the **"attention budget"**: every new token in context depletes some of it.

**2. Training-data distribution.** Pretraining corpora are dominated by shorter sequences — articles, code files, conversations, papers. The model develops most of its attention machinery on those length scales. Long-document training exists, but it's a much smaller fraction of total tokens. The model therefore has *less learned behavior* for very long contexts: fewer specialized attention heads, less practice connecting tokens at large distances.

**3. Position encoding extrapolation.** Models are trained with a fixed context window (say 8K or 32K). To handle longer windows in deployment, techniques like **RoPE scaling** and **position-encoding interpolation** stretch the position signals to cover the extended range. This works — the model doesn't error out — but the stretching introduces noise in token-position understanding. The model knows tokens are far apart but loses precision about *how* far.

The compound effect: a performance gradient, not a cliff. Models remain highly capable at longer contexts but show reduced precision for information retrieval and long-range reasoning compared to shorter contexts.

![Context rot — performance curve](../diagrams/svg/context-rot-curve.svg)

In benchmarks, the gradient looks roughly like the figure above. At 1K-8K tokens, recall is near-golden — these are the lengths the model was thoroughly trained on. At 32K-128K, performance is still strong but you start to see occasional retrieval failures, especially in the middle of the window ("lost in the middle"). At 200K-500K, needle-in-a-haystack tests start to fail noticeably — the model finds needles at the start or end of the window much more reliably than ones buried in the middle. At 1M+ tokens, attention dilution combines with position-encoding extrapolation noise and the gradient steepens; performance varies a lot between models.

The exact shape varies between models. Gemini 1.5 and 2.0 (which were architected with long context as a primary target) handle the curve more gracefully than older Claude/GPT variants. Reasoning models (o1, R1) can spend extra thinking tokens to *compensate* for context rot — but at a cost. Even the best frontier models exhibit *some* degradation.

### Specific named symptoms

- **Lost in the Middle** (Liu et al. 2023) — the U-shaped accuracy curve: information at the start or end of the context is recalled more reliably than information buried in the middle.
- **Distractor brittleness** — performance drops sharply when adding *plausible-but-irrelevant* content alongside the relevant content, even if the absolute relevant-token count is unchanged.
- **Self-distraction** — the model attends to its own earlier thoughts/tool results instead of the current task, especially in long agent loops.
- **Prompt drift** — the agent slowly forgets early instructions as the conversation grows.

### Why bigger windows don't fix it

It's tempting to wait for context windows to keep growing and assume the problem disappears. Anthropic's stance — and the empirical record from 2023-2026 — is that **bigger windows have not eliminated context rot, only shifted the curve**. As context windows scaled from 4K (GPT-3.5) to 32K to 200K to 1M to 2M tokens, each generation moved the rot threshold further out, but the shape of the curve persisted. Models trained specifically for long context perform better at 500K than older models at 200K, but they still degrade beyond their sweet spot.

The architectural reasons (quadratic attention, training distribution, position encoding) are fundamental to the transformer family. Some research directions (Mamba/SSMs, hybrid architectures, retrieval-augmented attention) attack these constraints directly, but the dominant production models in 2026 are still transformer-based and still experience context rot.

## Variants & related patterns

- [**Context Engineering — overview**](context-engineering-overview.md) — the discipline that arose specifically because of context rot.
- [**Compaction**](compaction.md) — the primary mitigation: summarize and restart.
- [**Just-in-Time Context**](just-in-time-context.md) — avoid the problem by not pre-loading.
- [**Progressive Disclosure**](progressive-disclosure.md) — load layer by layer; never fill the window upfront.
- [**Structured Note-Taking**](structured-note-taking.md) — externalize state so it doesn't compete for attention.
- **Sub-agent architectures** (see [`../agt/`](#)) — give each agent a fresh, focused context.
- **RAG vs long-context debate** — context rot is the empirical reason RAG hasn't been displaced even as windows grew.

## When NOT to use (i.e., when to ignore context rot)

Context rot is a fact about all transformer LLMs; you can't "not use" it. But you can choose to ignore it as a design concern in some cases:

- **One-shot summarization of a small document.** If you're summarizing a 5-page PDF, context rot isn't a problem — you're well within the high-recall zone.
- **Short agents.** If your entire agent loop comfortably fits in 20-30K tokens, you don't need compaction or structured note-taking. Adding that infrastructure has cost.
- **Single-question RAG with curated chunks.** If you fetch 5 relevant chunks and ask one question, rot isn't your bottleneck — chunk quality is.
- **Very simple structured-output tasks.** Classification, extraction, translation — short inputs, short outputs. Mostly immune.

Where you cannot ignore it: long-horizon agents (research, coding, multi-step automation), production agents serving real users, anything that loops 10+ times, anything with large tool definitions, anything with extensive message history.

## Implementations

Context rot is the *target* of a family of patterns rather than something you implement directly. Tools that help you measure or mitigate it:

| Tool | What it does |
|---|---|
| **RULER** (benchmark) | Measures effective context length across multiple tasks; identifies where models start to fail |
| **NoLiMa** (benchmark) | Needle-in-haystack with literal-match removed; tests true comprehension at long context |
| **BABILong** | Long-context reasoning benchmark from Berkeley |
| **Anthropic's needle-in-haystack tooling** (internal, methodology public) | Reference methodology — vary needle position and total context length |
| **OpenAI evals + your own NIH tasks** | Custom needle-in-haystack evals on your own corpus |
| **Compaction prompts** (see [`compaction.md`](compaction.md)) | The mitigation in your harness |
| **Tool result clearing** (Claude Developer Platform) | Drop raw tool outputs from history once consumed |

## Companies / projects acknowledging context rot

- **Anthropic** ✅ — explicitly named the phenomenon in the Sept 2025 post; designed Claude Code's compaction + memory tool around it. ([source](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents))
- **Google DeepMind** ✅ — Gemini 1.5 technical report (2024) explicitly discusses long-context retrieval degradation and architectural mitigations.
- **OpenAI** ⚠ — GPT-4-turbo and later released with long context but with public discussion of recall trade-offs; specific framing as "rot" not used.
- **Cognition (Devin)** ⚠ — devin.ai engineering posts discuss "context engineering" without using the term "rot."
- **HumanLayer / 12-Factor Agents** ✅ — Factor 3 framing builds on the same reality. ([source](https://github.com/humanlayer/12-factor-agents))

## Further reading

- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic, Sept 2025 (coined the term)
- [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172) — Liu et al. 2023 (foundational)
- [RULER: What's the Real Context Size of Your Long-Context Language Models?](https://arxiv.org/abs/2404.06654) — NVIDIA 2024
- [Greg Kamradt's needle-in-a-haystack methodology](https://github.com/gkamradt/LLMTest_NeedleInAHaystack) — the original benchmark format
- [Gemini 1.5 technical report](https://storage.googleapis.com/deepmind-media/gemini/gemini_v1_5_report.pdf) — Google's long-context architectural design

---

*Diagram source: [`../diagrams/src/context-rot-curve.d2`](../diagrams/src/context-rot-curve.d2), [`../diagrams/src/attention-budget.d2`](../diagrams/src/attention-budget.d2)*
