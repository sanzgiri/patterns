# Prompt Caching

**Aliases:** context caching, KV cache reuse, prefix caching, cached prompts, CAG (Cache-Augmented Generation)
**Category:** Memory
**Sources:**
[Anthropic — Prompt caching with Claude (Aug 14, 2024)](https://www.anthropic.com/news/prompt-caching) ·
[Anthropic — Prompt caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) ·
[Google — Gemini context caching (Feb 2024)](https://ai.google.dev/gemini-api/docs/caching) ·
[OpenAI — Prompt caching (Oct 1, 2024)](https://openai.com/index/api-prompt-caching/) ·
[Chan et al. — Don't Do RAG: When CAG Is All You Need (Dec 2024)](https://arxiv.org/abs/2412.15605)

---

## Problem

> [!TIP]
> **ELI5.** Most agent calls share a huge unchanging prefix — system prompt, tool definitions, retrieved docs, few-shot examples, maybe even a whole codebase. Without caching, the LLM provider re-processes those 10K+ tokens **every single call**, even though they're identical. That's expensive (you pay for input tokens) and slow (the model has to process them before generating). **Prompt caching** stores the model's internal computation (the KV cache) for the prefix on the provider side and reuses it on subsequent calls. Anthropic: up to **90% cost reduction** and **2× latency improvement** on the cached portion. Available from all three frontier providers as of late 2024. This is the single biggest cost optimization for agents in 2024-2026 — bigger than model choice, bigger than chunking strategy, often bigger than RAG vs long-context.

The mechanism is straightforward: large language models compute key-value (KV) tensors for every token in the input, then attend over them when generating the next token. For a fixed prefix, those KV tensors are *identical* across calls. Caching means: compute them once, persist them on the inference cluster, reuse them.

The economic impact was transformative for any application where calls share large prefixes:
- Coding agents (codebase context that doesn't change between turns)
- Customer support (KB + tools that don't change between tickets)
- RAG systems (tool definitions, system prompt, instructions)
- Document Q&A (the document itself)
- Few-shot heavy prompts (the examples)

Anthropic launched first (Aug 14, 2024), Gemini had context caching earlier (Feb 2024) with somewhat different semantics, OpenAI followed (Oct 1, 2024) with automatic caching. By 2026 all major inference providers support some form.

## How it works

> [!TIP]
> **ELI5.** Provider hashes the prefix of your prompt. If they've seen that exact prefix recently, they reuse the cached computation; you pay ~10% of the normal input-token price for the cached portion. If not, they compute and store it; you pay ~125% (the "cache write" premium) for that first call, then ~10% on every subsequent call within the cache TTL (5 min Anthropic, longer Gemini). The cache key is the **exact prefix bytes** — any change anywhere in the cached portion invalidates the cache.

![Prompt caching](../diagrams/svg/prompt-caching.svg)

### The math (Anthropic example)

Suppose you have a 10K-token system + tools prefix and you make 100 calls with different user messages:

**Without caching:**
- 100 calls × 10,003 input tokens = 1,000,300 input tokens billed
- All at the full rate (e.g., $3/MTok for Sonnet 4 → ~$3.00)

**With caching:**
- Call 1: 10,000 cache-write tokens (1.25× cost) + 3 user tokens at full rate
- Calls 2-100: 10,000 cache-read tokens (0.1× cost) + 3 user tokens at full rate
- Total billed cost ≈ Anthropic's cited ~10% of uncached cost

For coding agents with 50K-token codebase context and high call volume, this is the difference between "barely affordable" and "obviously affordable."

### Provider differences (late 2024 → 2026)

| Provider | Activation | TTL | Min prefix | Premium for write | Read price |
|---|---|---|---|---|---|
| **Anthropic** | Explicit `cache_control` breakpoints (up to 4) | 5 min default; 1 hr beta | 1024 tokens (Sonnet/Opus); 2048 (Haiku) | 1.25× input price | 0.1× input price |
| **Gemini** | Explicit cached content API; longer-lived | Hours-to-days configurable | 32K-128K (model dependent) | Standard input price | Varies; small per-token + per-hour storage |
| **OpenAI** | Automatic; no opt-in | ~5-60 min (load-dependent) | 1024 tokens | No premium | 0.5× input price (50% off) |

Anthropic and OpenAI both auto-route via prefix matching; Gemini requires you to *explicitly* create and reference a cache resource.

**Practical implication:** Anthropic gives you the deepest discount but you must structure prompts to put cacheable content first and mark breakpoints. OpenAI is most transparent (just works) but smaller discount. Gemini suits very long, very long-lived caches (whole books, whole codebases).

### Prompt structure for maximum cache reuse

The cache key is the exact prefix bytes. Anything that varies must come *after* anything that doesn't. The optimal structure:

```
[ system prompt + role definition    ]  <- cache breakpoint 1
[ tool definitions                   ]  <- cache breakpoint 2
[ large reference docs / codebase   ]  <- cache breakpoint 3
[ few-shot examples                  ]  <- cache breakpoint 4
--- end of cacheable region ---
[ recent conversation turns          ]
[ retrieved RAG chunks (if any)     ]
[ user's current message             ]
```

Common mistakes that invalidate the cache:
- Putting a timestamp in the system prompt
- Including the user's name early
- Changing the order of tools
- Adding `random_id` or session ID to the prefix
- Different whitespace / formatting between calls

### Cache hit rate is the key metric

Track cache hit rate per endpoint. A healthy production deployment runs >70-90% hit rate on long prefixes. If hit rate is low:
- Check that you're actually marking breakpoints (Anthropic)
- Check whether prefix varies (timestamps, IDs, etc.)
- Check TTL expiry — for sparse traffic, increase to 1-hour cache (Anthropic beta)
- Confirm that conversation history (which varies) is *after* static content

### Cache-Augmented Generation (CAG) — alternative to RAG

The [CAG paper (Chan et al. Dec 2024)](https://arxiv.org/abs/2412.15605) argues that for corpora that fit in the model's context window (now 200K-2M tokens), you should:
1. Load the entire corpus into the prompt once
2. Cache it
3. Answer queries against the cached corpus — no retrieval

Pros vs RAG:
- No retrieval misses (the model sees all of it)
- Holistic reasoning across the corpus
- Simpler architecture
- Cache hit is fast (90% cost reduction makes it competitive)

Cons:
- Only works for corpora that fit
- [Context rot](../ctx/context-rot.md) still applies — models attend less to mid-context content
- Costs scale linearly with corpus size (every query "sees" the whole corpus, even cached)
- Updates require cache invalidation + rebuild

**Sweet spot for CAG:** corpora 50K-500K tokens, queries that benefit from holistic context, stable content. Examples: a single legal contract, a textbook chapter, a project's docs + key code files.

### Cost engineering implications

Prompt caching changes the cost model dramatically:

- **Long-context becomes affordable.** A 1M-token Gemini call without caching is prohibitive; with caching it's competitive with RAG.
- **Few-shot prompting becomes cheap.** Heavy few-shot prompts (50 examples) used to be expensive per call; cached, they're nearly free.
- **Agent harnesses with big tool sets become viable.** 100 tools with detailed schemas? Cache them.
- **Multi-turn conversations get a free upgrade.** Conversation history can be cached up to the last user message.

### Anti-patterns

- **Random IDs in prefix.** Invalidates cache; defeats purpose.
- **Putting variable content first.** Forces a new cache entry per call.
- **Forgetting breakpoints (Anthropic).** Without `cache_control`, no caching happens.
- **Treating cache as guaranteed.** Cache can expire or be evicted. Code should work without it (just more expensive).
- **Caching small prefixes.** Below ~1024 tokens, the write premium outweighs the savings.
- **Caching very dynamic prompts.** If hit rate is <30%, caching loses money.
- **Not measuring hit rate.** Without telemetry, you don't know if caching is working.

### Production engineering

- **Telemetry**: log `cache_creation_input_tokens` and `cache_read_input_tokens` (Anthropic returns these). Compute hit rate.
- **Variants**: when running A/B tests on prompts, cache hit rate plummets — budget accordingly.
- **Stable serialization**: serialize tools, examples deterministically (same field order, same whitespace) — JSON dicts in different order = different cache key.
- **Choose TTL based on traffic**: high-frequency endpoints use default 5-min TTL; sparse endpoints benefit from 1-hour (Anthropic beta).
- **Cross-region caching**: caches are per-region; multi-region deployments have lower hit rates.
- **Cache-aware retries**: on transient errors, retry with same prefix to hit cache.
- **Avoid mid-cache writes**: don't append content into the middle of a cached region — only the suffix should vary.

### How prompt caching changes RAG vs long-context

Before caching: long-context was 100× more expensive than RAG, so RAG won for everything.

After caching: long-context with a hot cache costs about the same as RAG per query (sometimes cheaper, since no embedding lookup). RAG wins only when:
- Corpus too big to fit (>1-2M tokens)
- Corpus changes frequently (cache invalidates)
- You need fine-grained citation precision RAG provides

For many use cases that defaulted to RAG in 2023, CAG (long-context + prompt caching) is now the simpler and equally-cheap choice.

## Variants & related patterns

- [**Long-context vs RAG**](long-context-vs-rag.md) — the architectural decision caching enables.
- [**Memory architectures**](memory-architectures.md) — caching enables fat working memory.
- [**Context engineering overview**](../ctx/context-engineering-overview.md) — caching is a context-engineering tool.
- [**Compaction**](../ctx/compaction.md) — alternative when cache won't help.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — caching shifts the trade-off.
- [**Basic RAG**](../ret/basic-rag.md) / [**advanced RAG**](../ret/advanced-rag.md) — caching coexists with RAG.
- **KV cache** — the underlying mechanism inside transformers.

## When NOT to use

- **Short prompts** (<1024 tokens). The break-even isn't there.
- **Highly dynamic prompts**. Hit rate too low.
- **One-shot calls** with no repeated prefix.
- **When the prefix legitimately must vary** (per-request random sampling instructions, etc.).

## Implementations

| Provider | Implementation |
|---|---|
| **Anthropic Claude** | `cache_control: {type: "ephemeral"}` breakpoints (Aug 2024) |
| **Anthropic Claude (1-hour beta)** | Extended TTL for sparse-traffic endpoints |
| **OpenAI** | Automatic for prefixes ≥1024 tokens (Oct 2024) |
| **Google Gemini** | Explicit `CachedContent` API resource |
| **AWS Bedrock** | Anthropic + other models via Bedrock; prompt caching exposed |
| **Azure OpenAI** | Mirrors OpenAI behavior |
| **vLLM** | Native prefix caching (self-hosted) |
| **TGI (HuggingFace)** | Prefix caching support |
| **SGLang** | RadixAttention prefix caching (most aggressive) |
| **llama.cpp** | KV cache reuse |

## Companies / products that depend on prompt caching

- **Anthropic** ✅ — built and documented the feature.
- **OpenAI** ✅ — automatic prompt caching.
- **Google** ✅ — Gemini context caching.
- **Cursor, Continue, Cody, Aider** ⚠ — coding agents would be uneconomical without it.
- **GitHub Copilot Chat (Workspace)** ⚠ — likely uses prefix caching internally.
- **Glean, Notion AI, Slack AI** ⚠ — enterprise RAG with stable system prompts.
- **Most production agent frameworks** ⚠ — LangChain, LangGraph, Mastra, OpenAI Agents SDK all integrate.
- **Customer-support AI** ✅ — Intercom Fin, Zendesk AI use cached system context.
- **Anthropic's own Claude.ai** ⚠ — interactive chat caches system + conversation.

## Further reading

- [Prompt caching with Claude](https://www.anthropic.com/news/prompt-caching) — Anthropic Aug 2024 (canonical launch post)
- [Prompt caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — Anthropic reference
- [Gemini context caching](https://ai.google.dev/gemini-api/docs/caching) — Google docs
- [OpenAI prompt caching](https://openai.com/index/api-prompt-caching/) — Oct 2024
- [Don't Do RAG: When CAG Is All You Need](https://arxiv.org/abs/2412.15605) — Chan et al. Dec 2024
- [SGLang RadixAttention](https://arxiv.org/abs/2312.07104) — most aggressive open-source prefix caching
- [vLLM prefix caching docs](https://docs.vllm.ai/en/latest/automatic_prefix_caching/apc.html)

---

*Diagram source: [`../diagrams/src/prompt-caching.d2`](../diagrams/src/prompt-caching.d2)*
