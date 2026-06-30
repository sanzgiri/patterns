# Contextual Retrieval

**Aliases:** Contextual Embeddings + Contextual BM25, situated chunking, Anthropic-style RAG augmentation
**Category:** Retrieval
**Sources:**
[Anthropic — Introducing Contextual Retrieval (Sept 19, 2024)](https://www.anthropic.com/engineering/contextual-retrieval) (the canonical post) ·
[Anthropic Contextual Retrieval cookbook](https://github.com/anthropics/anthropic-cookbook/tree/main/skills/contextual-embeddings) ·
related: BM25 (Robertson & Zaragoza 2009), prompt caching, [Basic RAG](basic-rag.md)

---

## Problem

> [!TIP]
> **ELI5.** Imagine a chunk of text that reads "The company's revenue grew by 3% over the previous quarter." That chunk is useless on its own — you don't know *which company*, *which year*, or even what document it came from. When traditional RAG embeds this chunk, all that surrounding context is lost. A query like "What was ACME Corp's Q2 2023 growth?" won't match this chunk at all, because there's no "ACME" or "2023" anywhere in it. **Contextual Retrieval** fixes this by asking a cheap LLM (Claude Haiku) to write a short *situating* sentence for each chunk before embedding — "This chunk is from an SEC filing on ACME Corp's performance in Q2 2023..." — and prepending that sentence. Embeddings (and BM25 indices) are then built on the *contextualized* chunk. Result on Anthropic's benchmarks: **49% reduction in failed retrievals; 67% when combined with reranking**. It's the single best documented advance in RAG retrieval of 2024-2025.

The September 19, 2024 [Anthropic engineering post](https://www.anthropic.com/engineering/contextual-retrieval) by Christopher Olah and Anton Bakhtin is the canonical reference. The technique is conceptually simple but the benchmarks are striking — a one-time indexing cost yields a step-change in retrieval quality.

The problem it addresses, in Anthropic's own example: in an SEC-filings knowledge base, a chunk reading "The company's revenue grew by 3% over the previous quarter" loses its "which company, which quarter" context the moment it's split off from the document. Traditional RAG treats it as an isolated semantic unit. The query "ACME Corp Q2 2023 growth" simply can't match it — the embedder has no signal connecting those terms to this chunk.

The same problem afflicts most enterprise corpora — financial reports, legal documents, technical manuals, support tickets, scientific papers — where chunks routinely reference "this company," "the patient," "the protocol," "the system" with the actual referent in a different section.

## How it works

> [!TIP]
> **ELI5.** Three steps. (1) For each chunk in your corpus, send the *whole document* + the *chunk* to Claude Haiku with the prompt "give a short context that situates this chunk for search." Haiku returns ~50-100 tokens. (2) Prepend that context to the chunk text. (3) Embed the contextualized chunk *and* index it in BM25 — the contextualized chunk replaces the original everywhere it's stored. At query time: nothing changes. Just the index has been smartened up.

![Contextual Retrieval](../diagrams/svg/contextual-retrieval.svg)

### The exact prompt

Anthropic's published prompt for generating per-chunk context:

```
<document>
{{WHOLE_DOCUMENT}}
</document>

Here is the chunk we want to situate within the whole document
<chunk>
{{CHUNK_CONTENT}}
</chunk>

Please give a short succinct context to situate this chunk within
the overall document for the purposes of improving search retrieval
of the chunk. Answer only with the succinct context and nothing else.
```

That's it. The model is asked for a short context (typically 50-100 tokens), and that's prepended to the chunk verbatim before embedding.

### Why it works (and why it wasn't obvious)

The genius of contextual retrieval is that it bridges two worlds:
- **The world of indexing-time signal** — what you know about a chunk (its document, its section, its surrounding chunks) when you index it.
- **The world of query-time signal** — the user's natural-language query, which references things by name.

Without contextualization, those worlds don't align. The chunk says "the company"; the query says "ACME Corp." With contextualization, the chunk's embedding (and BM25 tokens) include "ACME Corp" explicitly, so they match the query.

Anthropic explicitly notes they evaluated alternatives that **don't** work as well:
- **Generic document summaries appended to chunks** — limited gains (the summary is too generic to help with specific queries).
- **Hypothetical document embeddings (HyDE)** — works for short queries but doesn't replace chunk-level context.
- **Summary-based indexing** — losses too much detail.

The specific combination — *per-chunk, situating, prepended, embedded together* — is what works.

### The benchmark numbers

From Anthropic's published evaluation (single source of truth):

| Setup | Reduction in retrieval failures vs basic RAG |
|---|---|
| Contextual Embeddings alone | ~35% |
| Contextual Embeddings + Contextual BM25 | **49%** |
| Contextual Embeddings + Contextual BM25 + Rerank | **67%** |

This is the single biggest retrieval improvement Anthropic has documented. The trick is that all the cost is at indexing time (one-time, cacheable), and query-time cost is unchanged.

### Cost analysis (the key practical insight)

The naive concern: "Doesn't this multiply indexing cost by 1000× if I need an LLM call per chunk?" The answer is "much less than you'd think," thanks to prompt caching.

The Anthropic prompt sends the *whole document* + each chunk. If a document has 100 chunks, you'd naively make 100 calls each sending the document — 100× the document tokens in inputs.

**With prompt caching** (Anthropic's cookbook covers this): the whole-document portion is cached after the first call; subsequent calls for chunks of the same document hit the cache at ~10% the cost. Quoted cost in the post: **~$1.02 per million tokens of contextualized chunks** using Claude 3 Haiku — cheap enough to be the default for any production RAG.

Practical implication: contextual retrieval is *one-time* indexing cost. Even for million-chunk corpora, it's a $1000-scale cost, not $100K. For most enterprise RAG deployments, the cost is rounding error compared to the quality gain.

### Implementation steps

1. **Use a small/fast model** (Claude 3 Haiku, Gemini Flash, GPT-4o-mini). The context-generation task is simple; don't overpay.
2. **Enable prompt caching** on the document portion. Order: `<document>{cacheable}</document>` first, then `<chunk>{varying}</chunk>`.
3. **For each chunk in a document**, run the prompt; capture the ~50-100 token context.
4. **Prepend** the context to the chunk text. Store the contextualized chunk as the source of truth for indexing.
5. **Embed** the contextualized chunk (any embedding model). Store vector in your vector DB.
6. **Index in BM25** (Elasticsearch, OpenSearch, or any sparse retrieval) — also using the contextualized chunk.
7. **Optionally cache the raw chunk** separately for display/citation; the contextualized version is for retrieval only.

### Querying — unchanged

At query time, **nothing changes**. The query is embedded the same way, BM25 the same way, hybrid fusion the same way, optionally reranked the same way. The improvement is purely at indexing time.

This is the underrated property of contextual retrieval — it's a **drop-in upgrade** to any existing RAG pipeline. No query-time latency cost, no API changes for the caller.

### Why it combines so well with hybrid + rerank

Anthropic's headline 67% reduction comes from stacking three techniques:
- **Contextual Embeddings** (dense) — captures semantic situation
- **Contextual BM25** (sparse) — captures exact keywords now present in chunk via context
- **Reranker** — final precision pass

Each stage independently helps; together they multiply. The interaction is favorable because:
- Contextual context adds named entities → BM25 has more keywords to match
- Reranker has higher-quality candidates to choose among (since contextualized retrieval already filtered better)
- No technique cannibalizes another's gain

### Where contextual retrieval doesn't help

- **Single-chunk documents.** If each "document" is one chunk, there's nothing to contextualize from.
- **Already-contextual chunks.** Some chunkers (semantic chunking by section, document-aware chunking) already preserve some context. The gain is smaller though usually still positive.
- **Tasks where the query *is* the keyword** ("error code TS-999") — BM25 alone catches these; contextual gains less.
- **Generic, non-named-entity corpora.** Pure how-to articles where "the system" is generic and ambiguous everywhere. Contextual still helps but less dramatically.

### When to skip contextual retrieval

- **Corpus fits in long-context window** (~200K tokens / 500 pages). Use prompt caching and just pass the whole thing.
- **Tiny corpora** where basic RAG already works well by inspection.
- **Real-time-changing corpora** where re-running indexing is too expensive. (Though incremental contextualization is straightforward.)

### Production engineering

- **Index versioning.** When you change the contextualization prompt, you need to rebuild the index. Version everything.
- **Cache hits.** Monitor prompt cache hit rate during indexing — should be high (>90%) for multi-chunk docs.
- **Per-chunk context length.** Anthropic suggests 50-100 tokens. Longer wastes embedding signal; shorter loses utility.
- **Quality eval.** Compare retrieval failure rate before/after on your own eval set. Anthropic's 49% number is for their benchmark; yours may differ.
- **Incremental updates.** New documents → contextualize their chunks. Updated documents → re-contextualize. Most pipelines handle this automatically.
- **Storage.** Both the raw chunk (for citation/display) and contextualized chunk (for embedding) should be stored. Slight storage overhead.
- **Citation behavior.** When citing back to user, cite the *original chunk* (without prepended context). The context is for retrieval only, not display.

### Variants and extensions

- **Multi-level context.** Some systems contextualize at section level + chunk level (document → section → chunk).
- **Domain-specific context generation.** Tune the context prompt for the domain (legal: cite case names; medical: cite patient/protocol).
- **Combine with metadata.** Sometimes the "context" is structured metadata (filename, section heading) — cheaper than LLM but less rich.
- **Re-contextualize on schema change.** If you add new metadata, regenerate.

### Anti-patterns

- **Skipping prompt caching.** Without it, indexing cost is 10× higher unnecessarily.
- **Using a strong model for context generation.** Haiku/Flash/Mini are plenty; don't pay for Opus.
- **Embedding the context separately and concatenating retrieval results.** Misses the point — the *combined* chunk needs to be the indexed unit.
- **Not eval'ing on your own data.** Anthropic's benchmarks are on their datasets; yours will differ. Always measure your own gain.
- **Loosing the original chunk text.** You need it for citations and human review.

### How it relates to other retrieval improvements

- **Stacks with [advanced RAG](advanced-rag.md)** — contextual retrieval is one technique in the broader advanced RAG toolkit.
- **Complementary to [reranking](reranking.md)** — best results combine both.
- **Alternative to [GraphRAG](graph-rag.md)** for relationship queries — GraphRAG is heavier; contextual retrieval is lighter and often sufficient.
- **Compatible with parent-document retrieval** — embed contextualized small chunks, return parent doc for context.
- **Pairs with [just-in-time context](../ctx/just-in-time-context.md)** philosophy — better retrieval enables more JIT.

## Variants & related patterns

- [**Basic RAG**](basic-rag.md) — the foundation contextual retrieval extends.
- [**Advanced RAG**](advanced-rag.md) — contextual retrieval is one of many techniques.
- [**Reranking**](reranking.md) — combines for 67% improvement headline number.
- [**GraphRAG**](graph-rag.md) — heavier alternative for relationship-heavy queries.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — RAG style philosophy.
- [**Structured note-taking**](../ctx/structured-note-taking.md) — situating context is analogous.

## When NOT to use

- **Corpus < 200K tokens.** Just pass the whole thing (with prompt caching).
- **Single-chunk docs** where there's no document context to draw from.
- **Real-time corpora** where re-indexing is operationally expensive (though incremental works).
- **Where you don't have an eval set.** Hard to know if it's helping.

## Implementations

| Tool | Contextual Retrieval support |
|---|---|
| **Anthropic Cookbook** | Reference implementation ([anthropic-cookbook/skills/contextual-embeddings](https://github.com/anthropics/anthropic-cookbook/tree/main/skills/contextual-embeddings)) |
| **LangChain** | Custom retrievers wrapping the technique |
| **LlamaIndex** | Custom transformations on chunks at ingestion |
| **Haystack** | Custom preprocessor |
| **Pinecone Assistant** | Production-grade support |
| **Vectara** | Native contextual chunking |
| **Custom** | The pattern is simple enough to implement directly |
| **Any vector DB + any LLM** | Works because indexing/query are unchanged |

## Companies / products using contextual retrieval

- **Anthropic** ✅ — published the technique ([source](https://www.anthropic.com/engineering/contextual-retrieval)).
- **Vectara** ⚠ — adopted similar contextualization in their RAG-as-a-service.
- **Glean** ⚠ — uses similar techniques in enterprise search.
- **Modal, Replicate, Together AI cookbooks** ⚠ — show contextual retrieval as default for production RAG.
- **Pinecone, Weaviate documentation** ⚠ — recommend the technique post Sept 2024.
- **Most enterprise RAG deployments in 2025-2026** ✅ — fast-becoming default.
- **Cursor, Continue, Cody (Sourcegraph)** ⚠ — contextualization of code chunks for repository RAG.

## Further reading

- [Introducing Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval) — Anthropic Sept 19, 2024 (canonical post; bookmark this)
- [Anthropic Contextual Embeddings cookbook](https://github.com/anthropics/anthropic-cookbook/tree/main/skills/contextual-embeddings) — runnable reference
- [Prompt caching with Claude](https://www.anthropic.com/news/prompt-caching) — the cost-enabler
- [BM25 reference](https://en.wikipedia.org/wiki/Okapi_BM25)
- [Retrieval-Augmented Generation Survey](https://arxiv.org/abs/2312.10997) — Gao et al. (broader context for RAG techniques)
- [What we learned from a year of building with LLMs](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — RAG quality lessons

---

*Diagram source: [`../diagrams/src/contextual-retrieval.d2`](../diagrams/src/contextual-retrieval.d2)*
