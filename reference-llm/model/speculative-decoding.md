# Speculative Decoding

**Aliases:** assisted decoding, predicted outputs, Medusa, EAGLE, lookahead decoding, draft-and-verify
**Category:** Model-level Patterns
**Sources:**
[Leviathan et al. — Fast Inference from Transformers via Speculative Decoding (2022)](https://arxiv.org/abs/2211.17192) (Google Research; canonical paper) ·
[Chen et al. — Accelerating LLM inference with speculative sampling (DeepMind 2023)](https://arxiv.org/abs/2302.01318) ·
[Cai et al. — Medusa: Simple LLM Inference Acceleration (2024)](https://arxiv.org/abs/2401.10774) ·
[OpenAI — Predicted Outputs (Nov 2024)](https://platform.openai.com/docs/guides/predicted-outputs) ·
[EAGLE paper (2024-2025)](https://arxiv.org/abs/2401.15077) ·
implementations: vLLM, TGI, llama.cpp, TensorRT-LLM, SGLang

---

## Problem

> [!TIP]
> **ELI5.** LLMs generate text one token at a time. Each token needs a full forward pass through the (huge) model. On a GPU, this is *memory-bandwidth bound*: most of the GPU's compute sits idle waiting to read the model weights. It's like driving a Ferrari in a school zone — engine underused. **Speculative decoding** fixes this with a clever trick: a small "draft" model proposes several tokens in advance; the big "target" model verifies all of them in ONE forward pass. If the draft was right, you got K tokens for the cost of 1. If wrong, you fall back to the big model's choice for that token. Output is mathematically identical to non-speculative decoding (same model, same temperature, same output distribution) — just faster. Standard 2-4× latency speedup in production today. Now in every major inference server (vLLM, TGI, TensorRT-LLM) and most API providers behind the scenes.

The mathematical insight ([Leviathan et al. 2022](https://arxiv.org/abs/2211.17192), [Chen et al. 2023](https://arxiv.org/abs/2302.01318)): for autoregressive decoding, the *bottleneck* isn't compute — it's reading the model weights from HBM (high-bandwidth memory) for each token. A forward pass with batch size 1 underutilizes the GPU dramatically. But a forward pass over MULTIPLE candidate tokens at once (verification) is nearly the same cost — same memory read, more compute per read.

So: have a small fast model propose K tokens; let the big model verify all K in one read; accept the prefix the big model agrees with; reject the rest. Math guarantees the output distribution is identical to standard decoding.

By 2026 the technique is standard. API users don't see it directly (it's behind the scenes) but it's what makes calls feel responsive. Self-hosters configure it explicitly via vLLM / TGI / TensorRT-LLM.

## How it works

> [!TIP]
> **ELI5.** A small model is fast but maybe slightly worse. The big model is slow but better. Use the small one to GUESS the next K tokens. Run the big model on (context + K guesses) — one pass. The big model produces its own probability for each guessed position. For each position in order: if the big model's choice agrees with the small model's, accept. As soon as you find a disagreement, replace that one token with the big model's choice and stop. You got the accepted prefix's worth of tokens for ONE big-model forward pass instead of K.

![Speculative decoding](../diagrams/svg/speculative-decoding.svg)

### The mechanics in detail

Given:
- **Target model** M (big, slow, what we want output from)
- **Draft model** m (small, fast, possibly the same family with fewer layers/params)
- **Lookahead K** (number of speculative tokens, typically 3-8)

For each generation step:

1. **Draft phase**: model m generates K tokens given context: `(t_1, t_2, ..., t_K)`.
2. **Verify phase**: model M runs one forward pass on `(context, t_1, t_2, ..., t_K)`, producing probabilities for the next token at each position.
3. **Accept/reject** in order:
   - For position 1: compare M's prob distribution to m's. Use rejection sampling: accept with probability `min(1, p_M / p_m)`. If accepted, keep `t_1`.
   - If rejected at position i: replace `t_i` with M's sampled choice; discard `t_{i+1}, ..., t_K`.
   - If all K accepted: accept all K plus M's choice for position K+1 (free bonus).
4. **Repeat** with updated context.

The rejection-sampling math is what guarantees output distribution equivalence with standard decoding. The expected number of accepted tokens per round depends on how often m agrees with M — typically 3-5 for well-chosen pairs on natural text.

### Why speedup is 2-4× (not K×)

You'd think K speculative tokens = K× speedup. Not quite, because:

- **Verification has cost** (same forward pass as one token, but slightly more compute).
- **Draft model takes time** (small but not zero).
- **Not all proposed tokens get accepted** — acceptance rate is often 40-70%.

Typical realized speedup: **2-3× on natural text, 3-5× on highly predictable text (code, structured output, translation)**.

### Variants

**Vanilla speculative decoding** (Leviathan, Chen 2022-2023). Uses a separate small model as drafter. Simplest, widely deployed.

**Medusa** ([Cai et al. 2024](https://arxiv.org/abs/2401.10774)). Instead of a separate drafter, add multiple prediction "heads" to the target model itself. Each head predicts a future token. Simpler infrastructure (one model), good acceptance rates.

**EAGLE** ([Li et al. 2024](https://arxiv.org/abs/2401.15077)). Uses target model's intermediate representations as features for a small draft model. Higher acceptance rate than vanilla; competitive with Medusa.

**Lookahead Decoding** ([LMSYS 2023](https://lmsys.org/blog/2023-11-21-lookahead-decoding/)). No draft model; speculate from the model's own n-gram history. Best for highly repetitive outputs (code, structured data).

**Predicted Outputs (OpenAI Nov 2024)**. The user passes in a "prediction" of what the output will be (e.g., the original code in a refactoring task). The API uses speculative decoding with the prediction as the draft. Free 2-4× speedup when applicable.

**Self-speculative decoding**. The target model drafts itself (using e.g. a pruned variant or early-exit). No separate draft model needed; competitive speedups.

**Multi-token prediction** (Meta 2024). Train a model to predict multiple tokens at once; use that as drafter.

### When speculative decoding works well

- **Code generation**: highly predictable tokens (keywords, brackets, common patterns) → high acceptance rate.
- **Structured outputs**: JSON, XML, SQL — predictable syntax.
- **Translation**: high redundancy in target language.
- **Summarization**: often paraphrases source text.
- **User-provided prediction**: refactoring with most of the output known.

### When it works less well

- **Creative writing**: low predictability → lower acceptance rate.
- **Reasoning model "thinking" traces**: highly variable; draft model struggles to predict.
- **Highly random sampling** (high temperature): randomness defeats prediction.
- **Long uncommon entities**: names, codes, identifiers that drafter doesn't know.

### Cost-quality trade-off

A key advantage: **output quality is unchanged.** Same model, same distribution, same outputs. Speculative decoding is *pure latency/throughput improvement*. It costs:

- **Extra compute** to run the draft model.
- **Slightly more memory** to hold both models.
- **Engineering complexity** to integrate.

But it costs nothing in quality — that's the magic. This is why it's a default in modern inference stacks.

### How to choose a draft model

For a target model M:

- **Same model family, fewer params**: e.g., Llama 3.1 70B + Llama 3.1 8B drafter. Most common.
- **Same model, fewer layers**: layer-pruned variant of M.
- **Distilled small model**: trained to mimic M's distribution.
- **N-gram heuristic**: not even a model; just predict based on recent tokens (good for repetitive content).

Choose drafter to maximize: acceptance rate × draft speed.

### Production engineering

- **vLLM, TGI, TensorRT-LLM, SGLang all support speculative decoding** with configurable draft models.
- **Acceptance rate is the key metric** — monitor it; tune K and drafter.
- **Per-domain tuning**: code workloads might prefer Medusa; chat might prefer vanilla.
- **Disabled for short responses** — overhead exceeds gain.
- **Disabled for high-temp sampling** — drafter accuracy drops.
- **GPU memory tradeoffs**: draft model takes memory; balance with batch size.
- **Streaming compatibility**: works fine with streaming; users see tokens stream in faster.

### Why this is "model-level" not "harness-level"

Unlike most app-developer patterns, speculative decoding lives *inside* the inference server (or API). Application developers typically don't choose to use it — it's enabled by:

- API provider's infrastructure (Anthropic, OpenAI, Gemini all use it server-side).
- Self-hosted inference server (vLLM, TGI).
- Inference SDK (TensorRT-LLM).

The exception: OpenAI's "Predicted Outputs" feature exposes the mechanism to application developers, letting them provide the draft directly. This is the only common case where app code interacts with speculative decoding.

### Anti-patterns

- **Mismatched draft/target distributions** — low acceptance rate; speedup negligible.
- **K too large** (16+) — overhead exceeds benefit.
- **K too small** (1-2) — minimal speedup.
- **Speculative decoding with temperature 2.0** — high randomness defeats prediction.
- **No acceptance-rate monitoring** — can't tune what you don't measure.
- **Speculative decoding for tiny outputs** — overhead exceeds benefit.

### Trajectory through 2026

- **Training-aware speculation**: models trained from start with speculative-decoding-friendly properties.
- **Hardware support**: GPUs and inference accelerators with dedicated speculative-decoding paths.
- **Per-request adaptive K**: tune speculation depth on the fly based on observed acceptance.
- **Multi-draft speculation**: try multiple draft sequences in parallel; pick the best-accepted.
- **Speculative decoding for reasoning models**: an active research area; current acceptance rates are poor for thinking traces.

For app developers: this remains mostly invisible infrastructure. For inference platform operators and self-hosters: this is the single biggest cost/latency optimization available.

## Variants & related patterns

- [**Brain vs hands**](../eng/brain-vs-hands.md) — different tier-routing strategy; complementary at app level.
- [**Reasoning models**](reasoning-models.md) — speculative decoding works less well on them.
- [**Prompt caching**](../mem/prompt-caching.md) — different optimization, often stacks (cache prefix, speculative-decode output).
- [**Augmented LLM**](../fnd/augmented-llm.md) — speculation invisible at this level.
- **Continuous batching** — the other big inference-server optimization (PagedAttention etc.).
- **KV cache compression** — adjacent inference optimization.
- **Quantization** — another speed/cost optimization (different mechanism).

## When NOT to use

- **You're an API user** — your provider already does it; nothing to configure.
- **Tiny outputs** where overhead exceeds gain.
- **High-temperature sampling** with low acceptance rates.
- **Reasoning model traces** (acceptance is low).
- **Memory-constrained inference** where draft model can't fit.

## Implementations

| Inference platform | Speculative decoding support |
|---|---|
| **vLLM** | First-class; configurable drafter |
| **TGI (HuggingFace)** | Native support |
| **TensorRT-LLM** | Native, including Medusa, EAGLE |
| **SGLang** | Native |
| **llama.cpp** | Speculative + draft model support |
| **MLC-LLM** | Speculative decoding |
| **OpenAI API** | Server-side (also exposes Predicted Outputs) |
| **Anthropic API** | Server-side (implicit) |
| **Google Gemini API** | Server-side (implicit) |
| **Modal, Together AI, Fireworks, Replicate** | Server-side, on by default |

## Companies / products benefiting from speculative decoding

- **OpenAI, Anthropic, Google** ⚠ — all use it server-side; users see lower latency.
- **GitHub Copilot** ⚠ — code completion uses speculative-style optimizations.
- **Cursor, Continue, Cody** ⚠ — code-mode latency benefits significantly.
- **Together AI, Fireworks, Anyscale, Modal** ✅ — productize inference with speculative decoding.
- **Self-hosting customers** (Llama, Qwen, Mistral users) ✅ — explicit configuration.
- **Anyone shipping low-latency LLM products** ✅ — speculative decoding is a default lever.

## Further reading

- [Fast Inference from Transformers via Speculative Decoding](https://arxiv.org/abs/2211.17192) — Leviathan et al. 2022 (canonical paper)
- [Accelerating Large Language Model Decoding with Speculative Sampling](https://arxiv.org/abs/2302.01318) — Chen et al. (DeepMind) 2023
- [Medusa: Simple LLM Inference Acceleration](https://arxiv.org/abs/2401.10774) — Cai et al. 2024
- [EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty](https://arxiv.org/abs/2401.15077) — Li et al. 2024
- [Lookahead Decoding](https://lmsys.org/blog/2023-11-21-lookahead-decoding/) — LMSYS
- [OpenAI Predicted Outputs](https://platform.openai.com/docs/guides/predicted-outputs)
- [vLLM speculative decoding docs](https://docs.vllm.ai/en/latest/usage/spec_decode.html)
- [TensorRT-LLM Medusa docs](https://nvidia.github.io/TensorRT-LLM/advanced/speculative-decoding.html)

---

*Diagram source: [`../diagrams/src/speculative-decoding.d2`](../diagrams/src/speculative-decoding.d2)*
