# Advanced RAG (Pre-retrieval, Hybrid, Post-retrieval)

**Aliases:** modular RAG, RAG v2, hybrid retrieval, query rewriting + reranking, RAG with HyDE/multi-query
**Category:** Retrieval
**Sources:**
[Gao et al. — Retrieval-Augmented Generation for Large Language Models: A Survey (2023-2024)](https://arxiv.org/abs/2312.10997) ·
[Gao et al. — HyDE: Precise Zero-Shot Dense Retrieval (2022)](https://arxiv.org/abs/2212.10496) ·
[Anthropic — Contextual Retrieval (Sept 2024)](https://www.anthropic.com/news/contextual-retrieval) ·
[Cormack et al. — Reciprocal Rank Fusion (2009)](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf) ·
production practice: Cohere, Voyage, Glean, Vectara playbooks

---

## Problem

> [!TIP]
> **ELI5.** [Basic RAG](basic-rag.md) (embed query → top-K chunks → LLM) breaks down on real-world queries. Users write short, ambiguous, jargon-heavy queries. Pure-vector search misses exact-match keywords like product names and error codes. Single-pass retrieval can't handle multi-hop questions. **Advanced RAG** stacks three layers of improvements: *fix the query before search* (rewriting, multi-query, HyDE), *use hybrid search* (dense + keyword, fused), and *fix the chunks after search* (rerankers, compression, diversity). Each layer is independently optional but most production systems use several. The 2024-2026 production RAG playbook is "basic RAG + 2-4 of these techniques," not "vanilla vector search."

The famous Gao et al. survey [Retrieval-Augmented Generation for Large Language Models](https://arxiv.org/abs/2312.10997) maps the modular RAG landscape: pre-retrieval, retrieval, and post-retrieval each have their own toolbox. The "naive RAG → advanced RAG → modular RAG" progression in that paper is the canonical reference.

Why basic RAG isn't enough in production:
- **Query-chunk vocabulary mismatch.** User asks "how do I fix `ERR_TIMEOUT`?"; the relevant doc says "request exceeded the configured deadline." Dense retrieval should catch this, but often doesn't for technical terms.
- **Top-K precision is low.** Of 10 retrieved chunks, often only 1-3 are actually relevant. Pollution dilutes generation quality.
- **Queries are too short to embed well.** "refunds" is too generic; the embedding is fuzzy.
- **Single-hop only.** "Who's the manager of the person who wrote this PR?" needs two retrievals.

Each technique below targets one of these failure modes.

## How it works

> [!TIP]
> **ELI5.** Think of advanced RAG as three layers around basic RAG. **Before search**: massage the query — rewrite it, expand it, generate a fake answer to embed (HyDE), or split it into sub-queries. **During search**: use BOTH dense (vector) AND keyword (BM25) search and combine results — they catch different things. **After search**: rerank the top-K with a more expensive but more accurate model, drop irrelevant sentences, diversify so you don't get 10 copies of the same chunk.

![Advanced RAG techniques](../diagrams/svg/advanced-rag.svg)

### Pre-retrieval techniques (improve the query)

**Query rewriting.** An LLM call to clean up the user's query before embedding it.
- Expands shortcuts ("Q4 nums" → "Q4 financial numbers")
- Removes filler ("can you please tell me about" → "...")
- Translates jargon to canonical form
- Cost: one extra LLM call. Worth it for ambiguous/short queries.

**Multi-query.** LLM generates K alternative phrasings of the user's query; retrieve for each; union the results.
- Captures different angles ("login error" vs "authentication failure" vs "can't sign in")
- Improves recall by K× at K× the search cost
- Works well with LangChain's MultiQueryRetriever pattern

**HyDE (Hypothetical Document Embeddings).** Ask the LLM to *hypothetically answer* the query, then embed the hypothetical answer (not the query) and search.
- Rationale: a hypothetical answer is much more semantically similar to relevant docs than a question is
- Surprisingly effective for short queries
- Failure mode: hallucinated hypothetical answer pulls in wrong-direction chunks

**Step-back prompting.** Ask the LLM to generate a more *general* version of the query first, retrieve for both general and specific, combine.
- Helps with detailed queries that have rich background context
- Useful for technical / scientific queries

**Query decomposition.** Split a complex query into sub-queries, retrieve for each, combine.
- "Compare X and Y" → "What is X?", "What is Y?", "How do they differ?"
- The basis of [agentic RAG](#) (where an agent plans the queries)

### Retrieval techniques (hybrid > dense alone)

**Hybrid search.** Run dense (vector) AND sparse (BM25/keyword) search in parallel; fuse the rankings.
- Dense catches semantic similarity; sparse catches exact keywords (product names, error codes, identifiers)
- The 2024-2026 production default in any serious RAG system
- Implementations: Elasticsearch hybrid, OpenSearch, Weaviate hybrid, Qdrant, pgvector + Postgres FTS

**Reciprocal Rank Fusion (RRF).** The standard way to combine rankings from multiple retrievers:
```
score(chunk) = sum_over_retrievers( 1 / (k + rank_in_retriever) )
```
Default k=60. Works without tuning, robust across retrievers. Cormack et al. 2009 original; still SOTA for hybrid fusion in many benchmarks.

**Metadata filtering.** Hard filter before vector search ("only docs updated in last 6 months", "only from `policy/` directory").
- Dramatically improves both precision and latency
- Critical for multi-tenant systems (filter by tenant)
- Failure mode: too restrictive filter → zero results

**Parent-document retrieval.** Embed small chunks for precision; return the larger parent chunk for context.
- "Search at sentence resolution, generate at paragraph resolution"
- LangChain ParentDocumentRetriever, LlamaIndex equivalents

**Auto-merging retrieval.** Multiple small chunks retrieved from the same parent → merge into the parent automatically.
- Reduces fragmentation in generation context

**Self-query retrieval.** LLM looks at the query and extracts both semantic search terms AND structured filters automatically.
- "Show me policy docs from 2024 about refunds" → vector query "refunds" + filter `type=policy AND year=2024`

### Post-retrieval techniques (improve the chunks)

**Reranking.** Take the top-N retrieved chunks (N=20-50) and run a more expensive but more accurate model to re-score them; keep only the top K (K=3-10). See dedicated [reranking](reranking.md) page.
- Biggest single quality lift in most production RAG systems
- Cross-encoder rerankers (Cohere Rerank, Voyage Rerank, BGE-reranker) are the standard
- LLM-as-reranker also viable for highest quality (most expensive)

**Compression / contextual compression.** Each chunk has irrelevant sentences. LLM (small, cheap) reads each chunk + query and extracts only relevant sentences.
- Reduces context pollution
- Tradeoff: extra LLM calls per chunk; usually worth it

**Diversity / MMR (Maximal Marginal Relevance).** Avoid retrieving multiple near-duplicate chunks.
- Re-rank by: (relevance to query) − λ × (similarity to already-selected chunks)
- Critical for content with high duplication (docs revised across versions)

**Context ordering.** Models attend better to chunks at the start and end of the context window (the "lost in the middle" effect, Liu et al. 2023). Put most-relevant chunks at the extremes; middle gets less attention.

**Chunk grouping by source.** Group all chunks from the same source document; present them together so the model can reason about a coherent source.

### Common production stacks

A typical 2024-2026 production RAG stack uses a subset, not all:

**Minimal-improvement (basic+):**
```
query → embed → hybrid search (dense+BM25, RRF fused) → reranker → top-5 → LLM
```
This alone often doubles quality over basic RAG.

**Mid-tier (most enterprise):**
```
query → query rewrite → embed → hybrid + metadata filter → reranker → MMR diversity → top-5 → LLM
```

**High-end (agentic):**
```
query → LLM decomposition → multi-query → hybrid + filter (per sub-query)
      → reranker → assembly → LLM with structured citations
```

### What to add when (decision tree)

- **Low recall** (relevant chunk missing from top-K) → multi-query, hybrid search, lower the K threshold
- **Low precision** (irrelevant chunks dominate) → reranker, MMR diversity, query rewriting
- **Multi-hop queries failing** → query decomposition, agentic RAG
- **Short queries failing** → HyDE, query rewriting
- **Exact-keyword failures** → hybrid (BM25 component)
- **Stale info problem** → metadata filter on date
- **Multi-tenant cross-leak** → metadata filter on tenant ID

### Cost vs quality

Each layer adds cost (LLM calls, more embeddings, more search latency):

| Technique | Extra cost | Quality gain (typical) |
|---|---|---|
| Hybrid search | 2× search latency, ~0 token cost | +15-25% recall |
| Reranker | 1 reranker call (~$0.001-0.01) | +20-40% precision |
| Query rewriting | 1 cheap LLM call | +5-15% on short queries |
| Multi-query | K× search cost | +10-20% recall |
| HyDE | 1 LLM call | +10-30% on short queries |
| Compression | 1 LLM call per chunk | +5-15% answer quality |
| Decomposition | 1+ LLM calls | Unlocks multi-hop |

Best ROI is almost always **hybrid search + reranker** — start there. Add others based on eval.

### Anti-patterns

- **All techniques at once.** Latency explodes, debugging gets hard. Add incrementally with eval.
- **No baseline.** Without measuring basic RAG first, you don't know whether the advanced techniques actually help.
- **HyDE on long queries.** HyDE helps with too-short queries; on detailed queries it's noise.
- **Multi-query without dedupe.** K queries → K × K chunks, lots of duplicates → context pollution.
- **Reranker on tiny top-N** (N=5). Reranker's value is choosing among many; needs N=20+.
- **Compression that drops citations.** Be careful that compression preserves source attribution.
- **Custom techniques before adopting the basics.** A surprising number of teams build novel retrieval before adopting hybrid+rerank.

### What's coming next (2025-2026)

- **Contextual retrieval** ([Anthropic Sept 2024](contextual-retrieval.md)) — add per-chunk context before embedding. The single biggest improvement of 2024-2025.
- **ColBERT-style late interaction** retrieval — token-level matching, expensive but very precise.
- **Hybrid with sparse-neural** (SPLADE, uniCOIL) — learned sparse retrievers competitive with dense.
- **Agentic RAG** — LLM as orchestrator over multiple retrieval tools.
- **GraphRAG** ([dedicated page](graph-rag.md)) — for relationship-heavy queries.

## Variants & related patterns

- [**Basic RAG**](basic-rag.md) — the foundation.
- [**Contextual retrieval**](contextual-retrieval.md) — adds context to chunks before embedding.
- [**GraphRAG**](graph-rag.md) — relationship-heavy alternative.
- [**Reranking**](reranking.md) — dedicated treatment.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — RAG is the canonical JIT pattern.
- [**Prompt chaining**](../wf/prompt-chaining.md) — multi-step RAG is a chain.
- [**Routing**](../wf/routing.md) — routing among multiple retrievers (by source, by language, by domain).
- [**Orchestrator-workers**](../wf/orchestrator-workers.md) — agentic RAG with planning.

## When NOT to use

- **Tiny corpus.** Put it all in context; skip retrieval.
- **Simple FAQ over stable KB.** Basic RAG suffices.
- **Latency-critical paths.** Each technique adds time; budget carefully.
- **Without an eval set.** You can't tell if it's helping. Eval first; then add.

## Implementations

| Layer | Tools |
|---|---|
| **Hybrid search** | Elasticsearch (RRF native), OpenSearch, Weaviate (built-in), Qdrant, pgvector + tsvector, Milvus |
| **Rerankers** | Cohere Rerank 3, Voyage Rerank 2, Jina Reranker, BGE-reranker, ColBERT, Mixedbread |
| **Query rewriting / HyDE** | Custom LLM calls; LangChain `MultiQueryRetriever`, LlamaIndex `HyDEQueryTransform` |
| **Compression** | LangChain `ContextualCompressionRetriever`, LlamaIndex `LongContextReorder` |
| **Hosted "advanced RAG"** | Vectara, Cohere RAG, Pinecone Assistant, Glean, OpenAI File Search (limited), Vertex AI Search |
| **Framework** | LangChain, LlamaIndex, Haystack, DSPy (programmatic) |

## Companies / products with advanced RAG

- **Glean** ✅ — enterprise search; hybrid + reranking + multi-source.
- **Perplexity, You.com** ⚠ — multi-query + reranking over web search.
- **Vectara, Cohere** ✅ — productize the full stack.
- **Pinecone Assistant** ✅ — hybrid + rerank out of the box.
- **GitHub Copilot Chat** ⚠ — hybrid retrieval over code (BM25 + semantic).
- **Notion AI Q&A, Slack AI Search** ⚠ — hybrid + reranking.
- **Cursor, Continue, Aider** ⚠ — hybrid retrieval over code repos.
- **Anthropic Claude.ai workspace** ⚠ — likely uses contextual retrieval + reranking.

## Further reading

- [Retrieval-Augmented Generation for Large Language Models: A Survey](https://arxiv.org/abs/2312.10997) — Gao et al. (canonical taxonomy)
- [HyDE: Precise Zero-Shot Dense Retrieval](https://arxiv.org/abs/2212.10496) — Gao et al. 2022
- [Reciprocal Rank Fusion](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf) — Cormack et al. 2009
- [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172) — Liu et al. 2023
- [Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) — Anthropic Sept 2024
- [Twelve RAG Pain Points and Solutions](https://towardsdatascience.com/12-rag-pain-points-and-proposed-solutions-43709939a28c/) — Wenqi Glantz
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank)
- [Pinecone hybrid search guide](https://docs.pinecone.io/guides/data/understanding-hybrid-search)

---

*Diagram source: [`../diagrams/src/advanced-rag.d2`](../diagrams/src/advanced-rag.d2)*
