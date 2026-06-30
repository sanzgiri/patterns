# Reranking (Cross-Encoder, LLM-as-Judge, ColBERT)

**Aliases:** two-stage retrieval, retrieve-then-rerank, cross-encoder reranking, LLM reranker, ColBERT late interaction
**Category:** Retrieval
**Sources:**
[Nogueira & Cho — Passage Re-ranking with BERT (2019)](https://arxiv.org/abs/1901.04085) (the original cross-encoder paper) ·
[Cohere — Rerank documentation](https://docs.cohere.com/docs/rerank) ·
[Voyage AI — Rerank documentation](https://docs.voyageai.com/docs/reranker) ·
[Khattab & Zaharia — ColBERT (2020)](https://arxiv.org/abs/2004.12832) ·
[Anthropic — Contextual Retrieval (Sept 2024)](https://www.anthropic.com/engineering/contextual-retrieval) reports +18% improvement from reranking ·
production practice: every serious enterprise RAG deployment 2024-2026

---

## Problem

> [!TIP]
> **ELI5.** Embedding-based vector search (the "first stage" in RAG) is FAST but only MODERATELY ACCURATE. To stay fast, the embedding model compresses each query and each chunk to one vector, computes similarity, and that's it. The query and chunk never "look at each other." A **reranker** is a second-stage model that takes the top-N (say 50) results from first-stage retrieval and re-scores each (query, chunk) pair *jointly* — it sees both at once, computes a much more accurate relevance score, and re-orders them. You then keep only the top-K (say 5) for generation. Reranking is the single highest-ROI quality lever in production RAG: adds ~50-200ms latency and a fraction of a cent per query, in exchange for *huge* precision gains (Anthropic's contextual retrieval post reported +18% reduction in retrieval failure from reranking alone).

The pattern goes by many names — "two-stage retrieval," "retrieve-then-rerank," "candidate-then-precision." All describe the same shape: a cheap retriever produces many candidates; an expensive but accurate model re-orders them. This is the same pattern as search engines have used for decades (BM25 → learning-to-rank → neural reranker). LLM-era RAG inherits it.

The core insight goes back to [Nogueira & Cho's 2019 cross-encoder paper](https://arxiv.org/abs/1901.04085): models that process query and document *together* (cross-encoders) score relevance dramatically more accurately than models that score them *separately* (bi-encoders, the architecture of embedding models). The price is computational — you can't pre-compute cross-encoder scores. So the trick is to apply the cross-encoder only to the top-N candidates from cheap retrieval.

## How it works

> [!TIP]
> **ELI5.** First stage retrieves 50 candidate chunks fast. Reranker takes each (query, chunk) pair, computes a relevance score by examining them *together* (not as separate vectors), and produces an ordered list. You keep the top 3-10 and send them to the generation LLM. Three reranker families exist: (1) cross-encoder rerankers (Cohere Rerank, Voyage Rerank, BGE-reranker) — small specialized models, fast enough for production. (2) LLM rerankers — use a general LLM with a scoring prompt, highest quality but expensive. (3) ColBERT — late-interaction models that hit a sweet spot between speed and quality.

![Reranking](../diagrams/svg/reranking.svg)

### Why bi-encoders are weaker than cross-encoders

The embedding model used for first-stage retrieval is a **bi-encoder**: it produces an embedding for the query *and* an embedding for each chunk *independently*, then computes similarity (typically cosine). Each side never sees the other.

A **cross-encoder** takes both the query and the chunk as a single input (`[CLS] query [SEP] chunk [SEP]`) and produces a single relevance score. Internal attention layers let every query token attend to every chunk token.

The cross-encoder is more accurate because it can capture interaction effects: "this chunk mentions X, but is it the same X the query asks about?" Bi-encoders compress this away. The trade-off: cross-encoders can't be pre-computed (the score depends on the query-chunk *pair*, not the chunk alone), so they're applied only at query time, only on the top-N candidates.

Production RAG combines both: bi-encoder (fast, pre-computed, high-recall) for first stage; cross-encoder (slow, query-time, high-precision) for second.

### Three reranker families

**1. Cross-encoder rerankers (the default).** Small (millions to a few billion params), purpose-built for relevance scoring.

- **Cohere Rerank 3.5** — leading commercial reranker, multilingual, ~100ms for 100 docs
- **Voyage Rerank 2** — top of recent benchmarks; supports very long context
- **Jina Reranker** — open API, competitive quality
- **BGE-reranker (v2-m3, v2-gemma)** — open-source, deployable
- **mxbai-rerank-large** — Mixedbread's open-source reranker
- **Cross-encoder/ms-marco-*** — classic SBERT-style models, free, self-host

These are the production default. Fast enough for real-time, accurate enough to justify the latency, costs in fractions of a cent per query.

**2. LLM-as-reranker.** Use a general-purpose LLM (Claude, GPT, Gemini) with a scoring prompt.

- Pass the query + a numbered list of candidate chunks; ask the LLM to return them re-ranked
- Highest quality of any reranker family — the LLM understands nuance specialized rerankers miss
- Most expensive (full LLM inference per query)
- Highest latency (1-3s)
- Used selectively: high-stakes queries, complex reasoning, premium tiers

**3. Late interaction (ColBERT-style).** Token-level matching rather than chunk-level scoring.

- ColBERT, ColBERTv2, PLAID: per-token embeddings; query token interacts with each chunk token
- More expensive index (per-token vectors), but very accurate retrieval-AND-rerank in one
- ColBERTv2 + PLAID achieves cross-encoder-level quality at bi-encoder-level latency
- Production use: smaller; some research-grade and search-engine deployments
- Tools: RAGatouille (PyTorch wrapper), Vespa, Vespa Cloud

### When each family wins

| Need | Best choice |
|---|---|
| Default production RAG | Cross-encoder (Cohere/Voyage/BGE) |
| Highest-quality answers, cost permitting | LLM-as-reranker |
| Lowest latency at high recall | ColBERT family |
| Self-host, open-source | BGE-reranker / mxbai-rerank |
| Multilingual | Cohere Rerank or Voyage |
| Very long chunks (>10K tokens) | Voyage Rerank (long-context) |
| Code reranking | Voyage Rerank Code, or LLM reranker |

### Typical configuration

```
First stage:    hybrid search (vector + BM25, fused via RRF) -> top-50
Reranker:       Cohere Rerank 3.5 -> top-5
Generation:     pass top-5 chunks to LLM
```

Key parameters:
- **N (first-stage top)**: 20-100. Too small wastes the reranker; too large adds latency.
- **K (reranked top)**: 3-10 typically. K depends on generation context budget.
- **Cutoff threshold**: optionally drop chunks below a score threshold; "if nothing scores above 0.5, return 'no info'."

### Cost and latency

| Reranker | Latency (N=50) | Cost per query (typical) |
|---|---|---|
| Cohere Rerank 3.5 | ~50-100ms | ~$0.001 |
| Voyage Rerank 2 | ~50-100ms | ~$0.0005-0.002 |
| BGE-reranker (self-host) | ~50-200ms | infra cost only |
| ColBERT (self-host) | ~10-50ms | infra cost only |
| LLM reranker (Claude Haiku) | ~500-1500ms | ~$0.005-0.02 |
| LLM reranker (Claude Sonnet) | ~1-3s | ~$0.05-0.20 |

For most production RAG, cross-encoder reranking is the sweet spot: a fraction of a cent, minimal latency, big quality gain.

### Quality gains (the empirical case)

Reported gains from production RAG benchmarks:

- **Anthropic Contextual Retrieval post** ([Sept 2024](https://www.anthropic.com/engineering/contextual-retrieval)): reranking layered on contextual retrieval reduced retrieval failures from 49% to 67% — i.e., reranking alone contributed ~18 percentage points.
- **Cohere Rerank docs**: "average nDCG@10 improvement of 20-50% across customer benchmarks."
- **Voyage Rerank benchmarks**: 10-30% precision@5 improvement over bi-encoder baseline.
- **Internal eval sets across many companies**: reranking is consistently the highest-impact single addition.

Empirically: reranking is **always worth trying** if you're not yet using one. The exceptions (latency-critical paths) are narrow.

### Production engineering

- **Eval set is mandatory.** Different rerankers favor different query distributions. Test on your data.
- **First-stage tuning matters too.** A bad first stage limits what the reranker can do. Make sure first-stage recall is high (target recall@N ≥ 90%).
- **Track recall@N and precision@K separately.** Recall is first-stage's job; precision is reranker's job.
- **A/B test rerankers.** Cohere Rerank 3.5 may not beat BGE-reranker v2-m3 on your data; only the eval tells you.
- **Score calibration.** Different rerankers produce scores on different scales. If you use score thresholds, calibrate per reranker.
- **Latency budget.** Reranker adds 50-200ms. Acceptable for most apps; tight for streaming-first interfaces.
- **Fallback on reranker failure.** If reranker API errors, fall back to first-stage ordering (degraded but functional).
- **Cache reranker results** on identical (query, chunk-set) pairs — common for popular queries.
- **Per-query-type policy.** Some queries don't benefit much from reranking (lookup queries with one obvious answer); others benefit hugely (ambiguous queries). Route based on query analysis if you can.

### Common variations

- **Multi-stage reranking.** First-stage → cheap reranker (BGE) → top-20 → expensive reranker (LLM) → top-5. Used when budget allows for premium quality.
- **Query-dependent reranking.** Different rerankers for different query types (factual vs. analytical).
- **Diversity-aware reranking.** Combine reranker score with MMR-style diversity penalty to avoid 5 near-duplicates.
- **Conditional reranking.** Skip reranking when first-stage confidence is very high (the top result clearly dominates).
- **Pre-filtered reranking.** Apply metadata filters before reranking so you only rerank relevant candidates.

### Anti-patterns

- **Reranking 5 candidates.** Defeats the purpose; reranker's value is selecting from many.
- **No first-stage tuning.** A 70%-recall first stage caps the reranker at 70%.
- **Same reranker as embedder.** Need a different model architecture (cross-encoder vs bi-encoder) to add value.
- **LLM-as-reranker on every query.** Too expensive; reserve for high-stakes paths.
- **Using reranker score as absolute relevance signal.** Reranker scores are *relative*; calibrate before applying thresholds.
- **Ignoring multi-lingual mismatch.** Many rerankers are English-first; non-English queries may degrade.
- **No fallback path** when reranker API is unavailable.

### Why reranking matters MORE in 2025-2026

Two trends increase reranking's importance:

1. **Bigger chunks and corpora.** With contextual retrieval and longer chunks, the "right answer in top-K" problem gets harder. Reranking is the precision lever.

2. **Agentic RAG with many retrieval tools.** When an agent dispatches multiple retrieval queries, results need to be merged and ordered — reranking is the natural combiner.

3. **Long-context generation models.** "Just put more in context" works only up to [context rot](../ctx/context-rot.md). Reranking gives you the *best* K chunks, not just K chunks.

### What's coming next

- **Reranker-aware embedders.** Embedders trained with reranker in mind, optimizing the joint pipeline.
- **Smaller, faster cross-encoders** (sub-100M params, sub-50ms latency at N=100).
- **Multi-modal rerankers** (text + image, text + code).
- **Learned-to-rank with user feedback** in production loops.

## Variants & related patterns

- [**Basic RAG**](basic-rag.md) — the system reranking improves.
- [**Advanced RAG**](advanced-rag.md) — reranking is the post-retrieval star technique.
- [**Contextual retrieval**](contextual-retrieval.md) — combines with reranking for compounding gains.
- [**GraphRAG**](graph-rag.md) — reranking applies on top.
- **Learning to rank** (Liu 2009) — the classical foundation.
- **Cross-encoders vs bi-encoders** (Reimers & Gurevych 2019, SBERT paper).
- **Late interaction** (ColBERT, PLAID).
- **MMR diversity** — combines with reranker score.

## When NOT to use

- **Latency-critical paths** with very tight budgets (<100ms total).
- **First-stage already returns 1-3 obviously-relevant results.** No room to improve.
- **Very small corpora.** First-stage retrieval is already accurate enough.
- **No eval set.** Can't measure the gain.

## Implementations

| Reranker | Type | Best for |
|---|---|---|
| **Cohere Rerank 3.5** | Cross-encoder | Production default; multilingual |
| **Voyage Rerank 2 / Voyage Rerank Code** | Cross-encoder | Long context; code |
| **Jina Reranker v2** | Cross-encoder | API alternative |
| **BGE-reranker (v2-m3, v2-gemma)** | Cross-encoder | Open-source self-host |
| **mxbai-rerank-large** | Cross-encoder | Open-source self-host |
| **MS MARCO cross-encoders (SBERT)** | Cross-encoder | Classic, free, self-host |
| **ColBERTv2 + PLAID** | Late interaction | High-recall high-precision |
| **RAGatouille** | ColBERT wrapper | Easy ColBERT in PyTorch |
| **LLM (Claude/GPT/Gemini)** | Generative | Premium quality, expensive |
| **Vector DBs with native rerank** | Various | Pinecone, Weaviate, Qdrant integrations |

## Companies / products using rerankers

- **Anthropic** ✅ — explicitly recommends reranking in Contextual Retrieval post.
- **Cohere customers** ✅ — Rerank product is widely deployed.
- **Voyage customers** ✅ — including Anthropic-recommended embedder + reranker combo.
- **Pinecone Assistant** ✅ — reranker in the pipeline.
- **Glean** ⚠ — enterprise search with reranking.
- **Vectara** ⚠ — reranker as part of pipeline.
- **GitHub Copilot Chat** ⚠ — code reranking over retrieved snippets.
- **Cursor, Continue, Cody** ⚠ — code chunk reranking.
- **Perplexity, You.com** ⚠ — rerankers on top of web search.
- **Most enterprise RAG deployments 2024-2026** ✅ — reranking is the default.

## Further reading

- [Passage Re-ranking with BERT](https://arxiv.org/abs/1901.04085) — Nogueira & Cho 2019 (cross-encoder reranking)
- [Cohere Rerank docs](https://docs.cohere.com/docs/rerank)
- [Voyage Rerank docs](https://docs.voyageai.com/docs/reranker)
- [ColBERT: Efficient and Effective Passage Search](https://arxiv.org/abs/2004.12832) — Khattab & Zaharia 2020
- [ColBERTv2](https://arxiv.org/abs/2112.01488) — Santhanam et al. 2021
- [PLAID](https://arxiv.org/abs/2205.09707) — Santhanam et al. 2022
- [Introducing Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval) — Anthropic Sept 2024 (combines with reranking)
- [RAGatouille](https://github.com/AnswerDotAI/RAGatouille) — easy ColBERT
- [BGE reranker (FlagEmbedding)](https://github.com/FlagOpen/FlagEmbedding) — open-source

---

*Diagram source: [`../diagrams/src/reranking.d2`](../diagrams/src/reranking.d2)*
