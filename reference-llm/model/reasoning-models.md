# Reasoning Models (Test-Time Compute)

**Aliases:** thinking models, chain-of-thought-as-behavior, inference-time compute, RL-trained reasoning, o-series, R-series
**Category:** Model-level Patterns
**Sources:**
[OpenAI — Learning to reason with LLMs (Sept 12, 2024)](https://openai.com/index/learning-to-reason-with-llms/) (o1 release) ·
[OpenAI — o3 (Dec 2024) / o3-pro / o4-mini (2025)](https://openai.com/) ·
[Anthropic — Visible Extended Thinking (Feb 2025)](https://www.anthropic.com/news/visible-extended-thinking) ·
[DeepSeek — R1 release (Jan 20, 2025)](https://api-docs.deepseek.com/news/news250120) ·
[Google — Gemini 2.0 Flash Thinking (Dec 2024) / 2.5 Thinking (2025)](https://deepmind.google/technologies/gemini/) ·
[Alibaba — QwQ-32B (Nov 2024) / Qwen3 Thinking (2025)](https://qwenlm.github.io/blog/qwq-32b-preview/)

---

## Problem

> [!TIP]
> **ELI5.** Traditional LLMs respond token-by-token, fast — and you can squeeze better reasoning out of them with [chain-of-thought prompting](../fnd/reasoning-prompts.md) ("think step by step"). But CoT is a *prompt technique* — fragile, model-dependent, often skipped. **Reasoning models** flip this: they're TRAINED (via reinforcement learning on reasoning traces) to internally generate a long *thinking* sequence *before* answering. The reasoning is a *model behavior*, not a prompt instruction. The result: on hard math, code, science, and planning problems, reasoning models dramatically outperform their non-reasoning peers — at the cost of 5-100× more tokens per response and 10-second-to-minute latency. OpenAI's o1 (Sept 2024) was the public proof-of-concept; the entire frontier converged on reasoning-model variants by mid-2025. By 2026, every major model family ships a reasoning variant; choosing reasoning-on vs reasoning-off is a primary per-call decision.

OpenAI's o1 announcement [Learning to reason with LLMs](https://openai.com/index/learning-to-reason-with-llms/) (Sept 12, 2024) was the watershed moment. The model showed dramatic improvement on AIME (math competition), Codeforces (competitive programming), and GPQA (PhD-level science) — gains of 30-60 percentage points over GPT-4o on the same benchmarks. The trick: the model was RL-trained to produce long internal reasoning chains. OpenAI hid those reasoning tokens at first (showed only summaries) but the cost was visible — o1 used 5-100× more tokens per response than GPT-4o.

DeepSeek-R1 (Jan 20, 2025) was the next inflection point: an open-weights model matching o1 on reasoning benchmarks, with the *reasoning training methodology* (GRPO + rule-based rewards) publicly documented. This kicked off rapid replication: by mid-2025, virtually every model family shipped a reasoning variant — Anthropic's extended thinking, Google's Gemini Thinking, Alibaba's QwQ and Qwen3-Thinking, Meta's reasoning-Llama experiments.

By 2026 the question isn't "is reasoning useful?" — it's "when do I pay for reasoning vs when do I use a fast model?"

## How it works

> [!TIP]
> **ELI5.** Reasoning models are trained (not prompted) to think longer. RL teaches them: when given a hard problem, generate a long monologue exploring approaches, catching errors, backtracking, verifying steps — THEN produce the final answer. You pay for the thinking tokens (sometimes hidden, sometimes shown). You wait longer. On hard problems, the answer is dramatically better. On easy problems, the extra tokens are wasted.

![Reasoning models](../diagrams/svg/reasoning-models.svg)

### The model family (2024-2026)

| Vendor | Reasoning model(s) | Key dates |
|---|---|---|
| **OpenAI** | o1 → o1-pro → o3 → o3-mini → o3-pro → o4-mini | Sept 2024 - 2025 |
| **Anthropic** | Claude Sonnet 3.7+ / Opus 4+ with extended thinking | Feb 2025+ |
| **Google** | Gemini 2.0 Flash Thinking, Gemini 2.5 Pro/Ultra Thinking | Dec 2024+ |
| **DeepSeek** | DeepSeek-R1 (open-weights), R1-Distill variants | Jan 2025 |
| **Alibaba** | QwQ-32B, Qwen3-Thinking | Nov 2024+ |
| **xAI** | Grok 4 Thinking | 2025 |
| **Mistral** | Magistral | 2025 |
| **Meta** | Llama 4 reasoning variants | 2025 |

Almost every serious foundation model family ships a reasoning variant by 2026.

### How they're trained

The dominant recipe (publicly documented by DeepSeek):

1. **Base model** with strong general capability.
2. **Reasoning data**: collect math problems, code problems, scientific Q&A with verifiable answers.
3. **Reinforcement learning with rule-based rewards**: model generates long reasoning + final answer; verifier checks correctness; reward only for correct final answer (initially) — model learns *whatever* internal reasoning leads to correctness.
4. **Mode-collapse safeguards**: encourage diversity in reasoning approaches.
5. **Format learning**: model learns to wrap reasoning in `<thinking>` tags or equivalent.
6. **Iterative refinement**: rejection sampling, more RL, fine-tuning on best traces.

The key insight: **don't supervise the reasoning trace** (it's hard to know which trace is "correct"). Just reward correct final answers. The model figures out what reasoning works through trial and error in training.

### Visible vs hidden thinking

- **OpenAI o1**: thinking tokens are hidden from users; summaries provided.
- **OpenAI o3-pro**: similar.
- **Anthropic extended thinking**: visible to users (the "Visible Extended Thinking" launch was explicit about this — Feb 2025).
- **Gemini Thinking**: typically visible.
- **DeepSeek R1**: fully visible.

Pros of visible:
- Users can verify the reasoning is sound.
- Better for debugging and trust.
- Aligned with the [maker-checker](../agt/maker-checker.md) pattern (the thinking is the "maker" output).

Pros of hidden:
- Defends against jailbreaks via reasoning manipulation.
- Removes commercial moat exposure.
- Cleaner UX for non-technical users.

By 2026 the trend is toward visible thinking, with API consumers paying for the reasoning tokens.

### Cost and latency

Reasoning models are expensive:

| Aspect | Non-reasoning model | Reasoning model |
|---|---|---|
| Tokens per response | 500-2000 | 5,000-100,000 |
| Latency p50 | 1-5s | 15s-3min |
| Latency p99 | 10s | 5-15min |
| Cost per response | $0.005-0.03 | $0.10-5.00 |

The variance within "reasoning models" is wide. Easy problems may produce short thinking; hard problems may think for many minutes.

### When to use reasoning models

Reasoning models excel at:
- **Math** (proofs, multi-step calculation, optimization)
- **Code** (algorithmic problems, complex refactoring, debugging)
- **Scientific reasoning** (analysis, planning experiments)
- **Multi-step planning** (e.g., research agents, route planning)
- **Logic puzzles** (constraint satisfaction, deduction)
- **Strategic decisions** (where slow analysis matters more than speed)

Reasoning models are NOT a win for:
- **Simple lookups** (FAQ, definitions, formatting)
- **Translation / paraphrase** (no reasoning needed)
- **Summarization** (mostly compression, not deduction)
- **Creative writing** (creativity ≠ deductive reasoning)
- **Latency-critical paths** (the wait is too long)
- **High-volume cost-sensitive paths** (per-call cost too high)

This is the canonical [brain vs hands](../eng/brain-vs-hands.md) split: reasoning models are brain; fast models are hands.

### Effort levels

Most reasoning models expose a "thinking budget" or "effort level":

- **OpenAI o-series**: `reasoning_effort: "low" | "medium" | "high"`
- **Anthropic extended thinking**: `thinking: { budget_tokens: N }`
- **Gemini Thinking**: thinking budget config
- **DeepSeek R1**: control via system prompt

Higher effort = more thinking tokens = more cost + latency + (usually) better answer. Tune per use case.

### How reasoning models change application architecture

- **Tool use becomes deeper.** Reasoning models can plan complex tool sequences without external orchestration; the orchestration *is* the reasoning.
- **Agent loops can be flatter.** Where you used to have a [ReAct](../fnd/react.md) loop with many turns, a reasoning model may do equivalent work in one extended response.
- **Evaluator-optimizer changes shape.** Critique + revision can happen *within* the model's thinking, not via separate calls.
- **You pay more attention to thinking budgets** than to prompt tweaks for these calls.
- **You hit rate limits and timeouts** more often; cap per call.

### Anti-patterns

- **Reasoning models on simple tasks.** Wasted cost; user waits.
- **No effort-level control.** Always max-effort blows budget.
- **No timeout.** Some reasoning chains spin for many minutes.
- **No fallback.** Reasoning model fails or times out; no fallback to non-reasoning model = broken endpoint.
- **Naive prompt re-use.** Prompts engineered for GPT-4o may not be optimal for reasoning models (the model already reasons; don't double-prompt CoT).
- **Treating thinking tokens as free.** They aren't.
- **Trusting reasoning blindly.** Reasoning models can also confabulate confidently. Verify externally where possible.

### What reasoning models are NOT good at

Despite their gains, reasoning models have known limitations:

- **They still hallucinate.** Longer reasoning sometimes produces more confidently-wrong outputs.
- **They underperform on creative tasks** vs non-reasoning frontier models.
- **They don't reliably know when to stop thinking.**
- **They're worse at instruction-following nuances** (sometimes the reasoning leads them astray of explicit instructions).
- **They cost more for tasks where reasoning isn't needed.**

These are active research areas; expect rapid improvement.

### Production engineering

- **Timeout per call** (60s-10min depending on use case).
- **Per-tier cost cap** (reasoning calls run up real money fast).
- **Streaming the thinking** for long calls so users see progress.
- **Cancellation propagation** (user navigates away → kill the reasoning call).
- **Eval set on reasoning specifically** — don't extrapolate from non-reasoning evals.
- **Logging thinking tokens separately** for observability + debugging.
- **Fallback chain** — try reasoning, fall back to non-reasoning on timeout or error.
- **Route decision in code** — `is_hard_problem(query) ? reasoning_model : fast_model`.

## Variants & related patterns

- [**Reasoning prompts**](../fnd/reasoning-prompts.md) — the prompt-engineered predecessor (CoT, ToT, etc.).
- [**Brain vs hands**](../eng/brain-vs-hands.md) — reasoning models are the canonical "brain."
- [**Augmented LLM**](../fnd/augmented-llm.md) — reasoning models are a kind of augmentation.
- [**Maker-checker**](../agt/maker-checker.md) — visible thinking is the "maker" output.
- [**Agent loop**](../agt/agent-loop.md) — reasoning models can compress agent loops.
- [**Workflows vs agents**](../agt/workflows-vs-agents.md) — reasoning models blur the distinction.
- [**Routing**](../wf/routing.md) — route hard queries to reasoning model.
- [**Evaluator-optimizer**](../wf/evaluator-optimizer.md) — can be internalized by reasoning models.
- **Test-time compute scaling laws** (research direction).
- **Process reward models** (alternative training signal).

## When NOT to use

- **Simple lookups, formatting, classification.**
- **Latency-critical user-facing paths.**
- **Cost-sensitive high-volume paths.**
- **Creative writing / open-ended generation.**
- **When you can't tolerate variable latency.**

## Implementations

| Vendor | Reasoning model API |
|---|---|
| **OpenAI** | o1, o3 family via Responses / Chat Completions API |
| **Anthropic** | Sonnet 4+ / Opus 4+ with `thinking` parameter |
| **Google** | Gemini 2.5 Thinking via Vertex AI / Gemini API |
| **DeepSeek** | R1 via API + Together / Fireworks / etc. (open-weights) |
| **Alibaba Cloud** | Qwen3-Thinking via DashScope |
| **xAI** | Grok 4 Thinking |
| **Hosted open-source** | Together, Fireworks, OpenRouter for R1, QwQ, Magistral |
| **Self-host** | vLLM, SGLang, TensorRT-LLM support reasoning models |

## Companies / products using reasoning models

- **OpenAI** ✅ — pioneered with o1; ships throughout ChatGPT and API.
- **Anthropic** ✅ — extended thinking in Claude.ai and API.
- **Google** ✅ — Gemini Thinking across Workspace and Vertex.
- **DeepSeek** ✅ — R1 widely deployed via API and Together / Fireworks.
- **Perplexity, You.com** ⚠ — reasoning models for complex queries.
- **Cursor agent mode** ⚠ — reasoning model for hard refactors.
- **Cognition Devin** ⚠ — reasoning model for planning.
- **GitHub Copilot Workspace** ⚠ — reasoning for complex tasks.
- **Research / data / finance / legal AI** ⚠ — reasoning model is the brain.
- **Most production LLM apps in 2025-2026** ⚠ — at least *route some* traffic to reasoning models.

## Further reading

- [Learning to reason with LLMs](https://openai.com/index/learning-to-reason-with-llms/) — OpenAI Sept 2024 (o1 launch)
- [Visible Extended Thinking](https://www.anthropic.com/news/visible-extended-thinking) — Anthropic Feb 2025
- [DeepSeek-R1: Incentivizing Reasoning Capability via RL](https://arxiv.org/abs/2501.12948) — DeepSeek Jan 2025 (training recipe)
- [QwQ-32B technical blog](https://qwenlm.github.io/blog/qwq-32b-preview/) — Alibaba Nov 2024
- [Chain-of-Thought Prompting Elicits Reasoning](https://arxiv.org/abs/2201.11903) — Wei et al. 2022 (the prompt-engineered predecessor)
- [Process Reward Models](https://arxiv.org/abs/2305.20050) — Lightman et al. (training methodology)
- [OpenAI o-series API guide](https://platform.openai.com/docs/guides/reasoning)
- [Anthropic extended thinking docs](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)

---

*Diagram source: [`../diagrams/src/reasoning-models.d2`](../diagrams/src/reasoning-models.d2)*
