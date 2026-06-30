# Long-Context vs RAG (and CAG)

**Aliases:** the long-context debate, RAG-or-context, CAG (Cache-Augmented Generation), hybrid retrieval
**Category:** Memory
**Sources:**
[Liu et al. — Lost in the Middle (2023)](https://arxiv.org/abs/2307.03172) ·
[Gemini 1.5/2.5 long-context papers (2024-2025)](https://blog.google/technology/google-deepmind/google-gemini-next-generation-model-february-2024/) ·
[Anthropic — Contextual Retrieval (Sept 2024)](https://www.anthropic.com/engineering/contextual-retrieval) (the "if it fits, just include it" note) ·
[Chan et al. — Don't Do RAG: When CAG Is All You Need (Dec 2024)](https://arxiv.org/abs/2412.15605) ·
[Hsieh et al. — RULER: What's the Real Context Size? (2024)](https://arxiv.org/abs/2404.06654)

---

## Problem

> [!TIP]
> **ELI5.** You have a 200-page document (or a 50-file codebase, or a year of support tickets). Two ways to make an LLM answer questions about it. **(A) RAG**: chunk it, embed it, retrieve top-K relevant chunks per query, generate. **(B) Long-context**: stuff the whole thing into the prompt window every call. In 2022 this debate was settled — long-context was 100× too expensive. In 2024-2026 it's wide open: Gemini 2.5 has 2M tokens, Claude Sonnet 4 has 1M, prompt caching cuts cost ~90%. Long-context is suddenly competitive — but [context rot](../ctx/context-rot.md) means quality degrades long before you hit the limit. The right answer depends on size, query patterns, cost, and how much the corpus changes.

The classic answer was settled by capacity: contexts were 4K-32K tokens in 2023, so any corpus over a few thousand words required retrieval. RAG won by default.

Three things changed:

1. **Long context windows.** Gemini 1.5 launched with 1M tokens (Feb 2024); Gemini 2.5 reached 2M; Claude Sonnet 4 hit 1M; GPT-4.1 / GPT-5 family extended significantly. Most frontier models now offer ≥200K context.

2. **Prompt caching.** Anthropic (Aug 2024), Gemini context caching (Feb 2024), OpenAI (Oct 2024) reduced the per-call cost of long-context calls by 75-90%. The economics inverted.

3. **"Lost in the middle"** ([Liu et al. 2023](https://arxiv.org/abs/2307.03172)) and [context rot](../ctx/context-rot.md) findings demonstrated that effective context is much smaller than nominal context. Quality degrades on most models well before the limit.

Result: the long-context-vs-RAG question is now legitimately context-dependent rather than settled. Anthropic's [Contextual Retrieval](../ret/contextual-retrieval.md) post explicitly says: "If your knowledge base is smaller than 200,000 tokens (about 500 pages), you can just include the entire knowledge base in the prompt that you give the model, with no need for RAG."

## How it works

> [!TIP]
> **ELI5.** Three options: long-context (stuff it all in, cache it), RAG (chunk and retrieve), or hybrid (use long-context for hot stable content + RAG for the long tail). Pick based on **size** (fits or doesn't), **stability** (changes often or rarely), **query type** (lookup or aggregation), and **cost tolerance** (caching helps a lot here). The decision tree usually goes: fits in 200K + stable → long-context with caching. Bigger or unstable → RAG. Need both breadth and depth → hybrid.

![Long-context vs RAG](../diagrams/svg/long-context-vs-rag.svg)

### Long-context (with caching) — the CAG pattern

**Cache-Augmented Generation** is the explicit name for: put the entire corpus in the prompt, mark it as cacheable, query against it.

```
[ system prompt + corpus (cached, 100K tokens) ]
[ user query                                    ]
→ LLM answer
```

When it shines:
- Single document Q&A (a contract, paper, manual)
- Small codebases (~50 files of relevant code)
- Stable corpus that rarely changes
- Aggregation queries ("what are the themes across these 200 articles?")
- Multi-step reasoning that benefits from holistic view
- Comparative queries ("how does section X differ from section Y?")

CAG works because [prompt caching](prompt-caching.md) makes the marginal cost of "all that context" tiny on each query after the first. The first call writes the cache (~1.25× cost on Anthropic, no premium on OpenAI); subsequent calls within 5 min - 1 hour read the cache at 10% of normal cost.

### When long-context breaks down

Long-context fails when:

- **Corpus too big.** 2M tokens is ~5K pages. Many real corpora are 100M+ tokens.
- **[Context rot](../ctx/context-rot.md).** Quality degrades well before the limit. Around 30-50K tokens on most models, attention degrades; specific recall ([RULER benchmark](https://arxiv.org/abs/2404.06654)) drops sharply.
- **Cost without caching.** A 1M-token Gemini call without caching is prohibitive. Cache misses bite.
- **Changing data.** Each cache invalidation means re-processing the whole corpus.
- **Per-tenant isolation.** Different tenants → different caches → no shared amortization.
- **Multi-tenant cross-leak risk.** Stuffing tenant A's data into a prompt risks cross-bleeding to tenant B's queries.

### RAG — what it still does best

- **Massive corpora.** RAG scales to billions of chunks; long-context can't.
- **Changing data.** New docs flow into the embedding index incrementally.
- **Per-tenant filtering.** Metadata filter at retrieval time isolates data cleanly.
- **Citation precision.** RAG returns specific chunks → cite the exact source.
- **Cost predictability.** Cost per query bounded by K chunks, not corpus size.
- **Selective relevance.** Why pay to process 200K tokens for a query about one paragraph?

### The hybrid pattern (production default for 2025-2026)

Most production deployments use both:

```
[ cached hot context: system + tools + project docs (50K tokens) ]
[ retrieved RAG chunks for this specific query (5K tokens)       ]
[ user query                                                      ]
→ LLM answer with citations
```

This combines:
- **Cache efficiency** for the unchanging hot context
- **RAG scalability** for the long tail
- **Citation precision** from retrieved chunks
- **Holistic awareness** from the loaded context

GitHub Copilot Chat, Cursor, Cody all use variants of this. Cursor's `@codebase` is essentially this: cached project context + RAG over the rest.

### Decision matrix

| Situation | Recommended |
|---|---|
| Corpus < 50K tokens, stable | Long-context (no caching needed) |
| Corpus 50K - 500K tokens, stable | **CAG** (long-context + caching) |
| Corpus 500K - 2M tokens, mostly stable | CAG (Gemini) or hybrid |
| Corpus > 2M tokens | **RAG** required |
| Corpus changes hourly | RAG (caching won't help) |
| Multi-tenant, per-tenant data | RAG with per-tenant filter |
| Single-doc deep Q&A | Long-context |
| Aggregation across many docs | Long-context if fits; otherwise GraphRAG |
| Need exact citations | RAG (clearer provenance) |
| Latency-critical | RAG (smaller context = faster) |
| Cost-sensitive, high query volume | Depends — measure both |

### Cost comparison (rough 2025 numbers, single query)

For a 100K-token corpus, query that needs ~5K tokens of relevant content:

| Approach | Input tokens billed | Cost estimate |
|---|---|---|
| Long-context, no cache | 100K | ~$0.30 (Sonnet 4 input rate) |
| Long-context, cached hit | ~100K cache-read | ~$0.03 |
| RAG (top-10 chunks of 500 tok = 5K) | 5K | ~$0.015 |
| RAG + reranker | 5K + reranker call | ~$0.016 |
| Hybrid (50K cached + 5K retrieved) | 50K read + 5K | ~$0.025 |

For high-volume endpoints with stable corpora, CAG is competitive with RAG and operationally simpler. For massive or changing corpora, RAG wins on cost and scaling.

### Quality comparison

RAG quality is bound by retrieval recall: if the right chunk isn't in top-K, the answer is wrong. [Contextual retrieval](../ret/contextual-retrieval.md) + reranking + hybrid search gets recall@10 to ~90-95% in good production systems — still 5-10% miss rate.

Long-context quality is bound by [context rot](../ctx/context-rot.md): the model technically has all the info but attention dilutes with input length. Effective recall on 200K-token inputs is usually 70-90% depending on task and model. Better than 5 years ago; not perfect.

For factual Q&A where the answer is in a specific passage, RAG with reranking often outperforms long-context. For aggregation, synthesis, or comparison, long-context usually wins because RAG misses cross-chunk relationships.

### "Lost in the middle" effect

Liu et al.'s 2023 paper showed models attend better to content at the *start* and *end* of context. Content in the middle ~30-70% range is recalled less reliably.

Implications:
- Put the most-critical content at the start or end of the cached region
- Put the query at the very end
- Don't trust 200K-token recall uniformly — it's u-shaped

The 2024-2025 [RULER benchmark](https://arxiv.org/abs/2404.06654) measures "effective context length" rigorously. Most models' effective context is 30-50% of nominal, depending on task difficulty.

### How long-context changes RAG architecture

Even when you're "doing RAG," long-context changes the engineering:
- Bigger chunks become viable (8K-16K instead of 500-1K). Fewer retrievals needed.
- Parent-document retrieval becomes more common (embed small chunks, return large parents).
- Aggregation queries get a "fetch everything relevant, then long-context summarize" pattern.
- Reranking can return more candidates (K=20 instead of K=5) since context budget allows.

### Anti-patterns

- **Long-context without caching.** Burns money for no quality reason.
- **Long-context for tiny queries.** Sending 100K tokens to ask "what's the title?" is wasteful.
- **RAG for tiny corpora.** Over-engineering for content that fits.
- **Mixing per-tenant data in shared long-context cache.** Security risk and cache invalidation nightmare.
- **Assuming nominal context = effective context.** RULER says otherwise.
- **No eval set.** Can't compare RAG vs CAG vs hybrid without measurement.
- **Picking long-context "because it's modern."** RAG isn't dead; it's complementary.

### Production engineering

- **Measure both** with an eval set. Don't assume.
- **Telemetry**: track cache hit rate (for CAG), recall@K (for RAG), end-to-end latency, cost per query.
- **Cache TTL strategy**: high-traffic CAG endpoints use default TTL; sparse use longer TTL.
- **Hybrid orchestration**: load cached project context always; add retrieved RAG chunks conditionally.
- **Per-tenant isolation**: never share long-context cache across tenants.
- **Cache invalidation discipline**: when corpus updates, invalidate cleanly; don't serve stale.
- **Fallback paths**: when cache misses, fall back gracefully (or pre-warm aggressively).

### How this debate will likely evolve

Through 2026:
- **Long-context grows**: 10M-token windows are coming. More corpora become long-context-viable.
- **Caching gets smarter**: longer TTLs, automatic optimal breakpoints, cross-session sharing.
- **Effective context catches up**: model improvements (sparse attention, modified positional encodings) reduce context rot.
- **Hybrid stays default**: even with 10M windows, RAG won't disappear; the patterns will coexist.

The "RAG is dead" claims of 2024 were premature. The "long-context replaces everything" claims are also wrong. The patterns are complementary, and good products use both.

## Variants & related patterns

- [**Basic RAG**](../ret/basic-rag.md) — the alternative this compares against.
- [**Contextual retrieval**](../ret/contextual-retrieval.md) — extends RAG quality.
- [**Prompt caching**](prompt-caching.md) — enables long-context economics.
- [**Context rot**](../ctx/context-rot.md) — the quality limit on long-context.
- [**Context engineering overview**](../ctx/context-engineering-overview.md) — the framing.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — RAG-aligned philosophy.
- [**Compaction**](../ctx/compaction.md) — alternative for long sessions.
- [**Memory architectures**](memory-architectures.md) — broader storage framing.
- **CAG (Cache-Augmented Generation)** — the named pattern.
- **RULER benchmark** — effective context measurement.

## When NOT to use long-context

- **Massive corpora** (>2M tokens).
- **Frequently changing data.**
- **Highly precise citation requirements.**
- **Multi-tenant systems** where data must be isolated.
- **Cost-extreme deployments** where even cached cost matters.

## When NOT to use RAG

- **Tiny corpora** that fit.
- **Aggregation queries** where retrieval misses the whole picture.
- **You can't afford the engineering** (chunking, embedding, vector DB, reranker tuning).
- **Stable hot context** that long-context-with-cache handles better.

## Implementations

| Pattern | Tools |
|---|---|
| **CAG** | Anthropic Claude + cache_control; Gemini cached content; OpenAI auto-cache |
| **RAG** | LangChain, LlamaIndex, Haystack, vector DBs (Pinecone, Weaviate, etc.) |
| **Hybrid** | Custom orchestration; LangGraph; OpenAI Assistants File Search + system prompt |
| **Long-context-aware frameworks** | Mastra, Vercel AI SDK with caching helpers |

## Companies / products navigating this trade-off

- **Anthropic** ✅ — explicitly recommends long-context-with-cache for ≤200K corpora.
- **Google** ✅ — Gemini 2.5 + context caching pushes long-context hard.
- **GitHub Copilot, Cursor, Continue, Cody** ⚠ — hybrid: cached project context + RAG over codebase.
- **Notion AI, Slack AI, Microsoft 365 Copilot** ⚠ — hybrid per workspace.
- **Glean** ⚠ — RAG-dominant (corpora too large for long-context).
- **Perplexity** ⚠ — RAG for web; long-context for follow-ups.
- **Box AI** ⚠ — long-context for single-doc Q&A; RAG for multi-doc.
- **Anthropic Projects, OpenAI custom GPTs** ⚠ — long-context for project files + RAG for File Search.

## Further reading

- [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172) — Liu et al. 2023
- [RULER: What's the Real Context Size of Your Long-Context Language Models?](https://arxiv.org/abs/2404.06654) — Hsieh et al. 2024
- [Don't Do RAG: When Cache-Augmented Generation Is All You Need](https://arxiv.org/abs/2412.15605) — Chan et al. Dec 2024
- [Introducing Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval) — Anthropic Sept 2024 (the "if it fits, just include it" framing)
- [Prompt caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — Anthropic
- [Gemini long context documentation](https://ai.google.dev/gemini-api/docs/long-context)
- [Anthropic — How to think about long context](https://docs.anthropic.com/en/docs/build-with-claude/context-windows)

---

*Diagram source: [`../diagrams/src/long-context-vs-rag.d2`](../diagrams/src/long-context-vs-rag.d2)*
