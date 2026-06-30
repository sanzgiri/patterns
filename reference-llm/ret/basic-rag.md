# Basic RAG (Retrieval-Augmented Generation)

**Aliases:** vanilla RAG, naive RAG, embed-and-retrieve, vector search + LLM, RAG pipeline
**Category:** Retrieval
**Sources:**
[Lewis et al. — Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks (2020, original RAG paper)](https://arxiv.org/abs/2005.11401) ·
[Anthropic — Introducing Contextual Retrieval (Sept 2024)](https://www.anthropic.com/news/contextual-retrieval) ·
[OpenAI Cookbook — RAG patterns](https://cookbook.openai.com/) ·
[LangChain RAG documentation](https://python.langchain.com/docs/tutorials/rag/) ·
production practice: every enterprise LLM deployment 2023-2026

---

## Problem

> [!TIP]
> **ELI5.** LLMs only know what was in their training data. They don't know your company's docs, your customer's tickets, or anything that happened last week. **RAG** is the answer: at query time, *retrieve* a few relevant chunks of your own docs, *stuff them into the prompt*, and let the LLM answer using them. The model essentially gets a focused open-book exam instead of a closed-book one. Result: answers grounded in your actual data, citations to specific sources, and the ability to update knowledge by updating the index instead of retraining. RAG was THE dominant LLM application pattern of 2023-2025 and is still foundational in 2026 even as long-context and agentic patterns evolve around it.

The original RAG paper ([Lewis et al. 2020](https://arxiv.org/abs/2005.11401)) introduced the architecture for knowledge-intensive NLP tasks. The 2022-2023 LLM explosion made it the dominant application pattern: virtually every "Chat with your docs" / "AI search" / "enterprise assistant" product is RAG underneath.

The problem RAG solves:
- **LLMs don't know your private data.** Training cuts off; your docs aren't in it.
- **Fine-tuning is expensive** and doesn't reliably "teach" facts (it teaches *style* and *behavior* better than recall).
- **Context windows aren't infinite** (even with 1M+ token models, [context rot](../ctx/context-rot.md) degrades performance long before you hit the limit).
- **You need citations.** Users want to verify; regulators require auditability.

RAG addresses all four: pull *only the relevant chunks* at query time, fit them in the context, generate with citations.

## How it works

> [!TIP]
> **ELI5.** Two phases. **Indexing** (offline): chop your docs into chunks, convert each chunk to a vector (a list of numbers that captures meaning), store in a database. **Querying** (per request): convert the user's question to a vector with the *same* embedding model, find the K nearest chunks by vector distance, put those chunks in the prompt along with the question. The LLM answers using those chunks. The same-model rule is critical — query and chunk vectors only compare meaningfully if they came from the same embedder.

![Basic RAG](../diagrams/svg/basic-rag.svg)

### Indexing phase (offline, periodic)

1. **Ingest documents.** PDFs, HTML, markdown, database rows, support tickets — whatever the source material is.
2. **Parse and extract.** Convert each document to plain text plus metadata (title, URL, date, section, author).
3. **Chunk.** Split text into overlapping segments (typical: 200-1000 tokens with 50-200 token overlap). Chunk size determines what "one retrieved unit" looks like.
4. **Embed.** Pass each chunk through an embedding model (OpenAI `text-embedding-3-small`/`-large`, Cohere `embed-v4`, Voyage `voyage-3`, etc.). Output: a 256-3072 dim vector per chunk.
5. **Store.** Persist `(vector, chunk text, metadata)` triples in a vector database (Pinecone, Weaviate, Qdrant, pgvector, Elasticsearch, etc.).

### Querying phase (per user request)

1. **Receive query.** "What's our refund policy for international orders?"
2. **Embed query.** Same embedding model as the index — this is mandatory. Different models produce vectors in incomparable spaces.
3. **Vector search.** Find K chunks with the highest cosine similarity (or other distance metric) to the query vector. Typical K = 5-20.
4. **Optional: metadata filter.** "Only chunks from policy docs updated in last year."
5. **Assemble prompt.** Pre-prompt + retrieved chunks + user query + instructions ("answer using only these chunks; cite source").
6. **LLM generation.** Pass the assembled prompt to the LLM; it generates an answer grounded in the chunks.
7. **Return answer with citations.** Often includes which chunks were used and which doc they came from.

### Why it works (the embedding insight)

Embeddings convert text into vectors where *semantic similarity* corresponds to *vector proximity*. A chunk about "refund policy" and a question "how do refunds work?" will have vectors near each other, even if they don't share keywords. This makes retrieval more robust than keyword search (BM25) for natural-language queries.

But — and this is the key 2024-2026 lesson — **pure dense embedding retrieval has blind spots**:
- It misses exact-keyword matches (product names, error codes, identifiers).
- It struggles with negation ("docs that *don't* mention X").
- It can retrieve semantically-related-but-wrong chunks.

That's why production systems graduated from pure-vector basic RAG to **hybrid retrieval** ([advanced RAG](advanced-rag.md)), where dense and keyword search complement each other.

### The minimal RAG prompt

A surprisingly small system prompt anchors RAG generation:

```
Answer the user's question using ONLY the provided context.
If the answer isn't in the context, say "I don't have that information."
Cite sources by chunk number.

Context:
[1] {chunk_1}
[2] {chunk_2}
...
[K] {chunk_K}

Question: {user_query}
```

Two failure modes to guard against:
- **Hallucination on missing info.** The "if not in context, say so" instruction matters. Without it, the model fills gaps with plausible-sounding fiction.
- **Ignoring context.** The model occasionally goes off on what it "knows." Few-shot examples of grounded answers + post-hoc citation verification reduce this.

### Chunking — the underrated lever

Chunk size and strategy hugely affect retrieval quality:

- **Too small** (50-150 tokens): chunks lack context to be useful; you retrieve fragments.
- **Too large** (>2000 tokens): chunks dilute relevance; retrieval matches on the *average* topic of a chunk, not the specific point.
- **Sweet spot** for most use cases: **300-800 tokens with ~100 token overlap.**

Beyond size:
- **Semantic chunking.** Split at paragraph / section boundaries, not arbitrary token counts.
- **Recursive chunking.** Try paragraph splits first, fall back to sentence splits.
- **Document-aware chunking.** Preserve heading hierarchy so chunks know their context.
- **Parent-document retrieval.** Embed small chunks, but at retrieval time return the larger surrounding document for full context.

The 2024 Anthropic [Contextual Retrieval](contextual-retrieval.md) work showed that *adding context to each chunk before embedding* (e.g., "This chunk is from Section 4 of the Q3 financial report, discussing revenue") improves retrieval by 35-50%. This is now the recommended default.

### Embedding model choice

By 2026 the top embedders:
- **OpenAI** `text-embedding-3-large` (3072d), `text-embedding-3-small` (1536d) — workhorse defaults.
- **Cohere** `embed-v4` — strong on multilingual and multi-modal.
- **Voyage** `voyage-3`, `voyage-3-lite` — top of MTEB for many tasks.
- **Anthropic** does not produce embeddings; they recommend Voyage.
- **Open-source**: `BGE` family, `E5`, `nomic-embed` — competitive and self-hostable.

Pick on:
- **MTEB benchmark** for general quality.
- **Domain specificity** — embedders trained on code, biomedical, legal vary in quality on those domains.
- **Dimensionality** — bigger isn't always better; trades cost vs precision.
- **Cost and latency** — embedding millions of chunks adds up.
- **Self-host vs API** — data sensitivity often forces self-host.

### What basic RAG is bad at

Listing the limitations is the bridge to [advanced RAG](advanced-rag.md):

- **Ambiguous queries** ("show me the Q4 numbers" — Q4 of what year?). Basic RAG returns nearest-by-meaning chunks; doesn't ask for clarification.
- **Multi-hop questions** ("what's the email of the CEO of our top customer?" — needs two retrievals, joined).
- **Aggregation** ("summarize all support tickets about login issues this month").
- **Comparative** ("how is our policy different from competitor X?").
- **Counterfactual / negation** ("which products do *not* support feature Y?").

Each of these motivates an extension in advanced RAG: query rewriting, multi-hop retrieval, agentic RAG, hybrid search, etc.

### When basic RAG is enough

A surprising amount of production traffic is well-served by basic RAG with no advanced techniques:
- Single-document Q&A ("what does this contract say about termination?").
- Documentation search ("how do I use this API?").
- FAQ-style customer support over a stable knowledge base.
- Simple "chat with your PDFs" use cases.

Start with basic RAG, measure quality (eval set!), and add complexity only when measurements show specific gaps. Premature jump to GraphRAG / agentic RAG is a common waste.

### Production engineering

- **Eval set is mandatory.** A few hundred (query, expected-chunks, expected-answer) triples. Without it, you can't tell whether changes help.
- **Track recall@K** for chunks: of the chunks needed to answer correctly, how many are in top-K?
- **Track answer quality** separately with [evals](../qua/eval-driven-development.md): grounded? cited? complete?
- **Latency budget.** Embedding + search + generation can be 1-3s. Cache embeddings of common queries.
- **Cost telemetry.** Embedding cost (one-time per doc) vs vector DB cost (storage) vs generation cost (per query).
- **Index refresh.** New/updated docs need re-embedding. Plan for incremental updates.
- **Citation verification.** Post-hoc check that cited chunk numbers actually appear in the prompt; flag hallucinated citations.
- **Drift monitoring.** Distribution of retrieved chunks should be stable. Sudden shifts indicate either content changes or embedder issues.

### Anti-patterns

- **No eval set.** "It works for me on 3 queries" doesn't generalize.
- **Different embedder for index and query.** Vectors aren't comparable; retrieval is random.
- **Massive chunks** (>2000 tokens). Retrieval becomes coarse.
- **Tiny chunks** (<150 tokens). Lose context; need many more chunks per answer.
- **No metadata.** Can't filter, can't cite usefully, can't update incrementally.
- **K too small** (K=1): unforgiving on retrieval errors. K=5-10 typical.
- **K too large** (K=50+): pollutes context with irrelevant material; [context rot](../ctx/context-rot.md) kicks in.
- **Trusting the LLM to refuse when context is bad.** Models hallucinate confidently; need grounded-output guardrails.

## Variants & related patterns

- [**Advanced RAG**](advanced-rag.md) — hybrid retrieval, query rewriting, HyDE.
- [**Contextual retrieval**](contextual-retrieval.md) — Anthropic Sept 2024; biggest single retrieval-quality improvement.
- [**GraphRAG**](graph-rag.md) — for relationship-heavy queries.
- [**Reranking**](reranking.md) — improves precision after retrieval.
- [**Just-in-time context**](../ctx/just-in-time-context.md) — RAG is the canonical JIT pattern.
- [**Augmented LLM**](../fnd/augmented-llm.md) — retrieval is one of the three augmentations.
- [**Prompt chaining**](../wf/prompt-chaining.md) — RAG is often a chain (retrieve → generate).
- **Long-context vs RAG** — alternative for whole-doc QA (use both: long-context for breadth, RAG for precision).
- **CAG (Cache-Augmented Generation)** — long-context with prompt caching, a 2024-2025 alternative.

## When NOT to use

- **The whole corpus fits in context.** Just pass it directly; no need for retrieval overhead. Long-context models make this viable for documents <500 pages.
- **The question doesn't need outside info.** General reasoning, code generation, creative tasks.
- **Aggregation tasks.** "How many tickets mentioned X this week?" is better served by SQL or a structured query, not RAG.
- **When the answer needs *all* documents.** RAG retrieves K chunks; if the answer requires synthesizing 1000s, use a different architecture.

## Implementations

| Component | Options |
|---|---|
| **Vector DB** | Pinecone, Weaviate, Qdrant, Chroma, Milvus, pgvector, Elasticsearch, OpenSearch, MongoDB Atlas Vector, Turbopuffer |
| **Embedding APIs** | OpenAI, Cohere, Voyage, Mistral, Google, AWS Titan |
| **Open-source embedders** | BGE, E5, nomic-embed, gte, Jina, ColBERT |
| **Framework** | LangChain, LlamaIndex, Haystack, LlamaParse, Mastra, DSPy |
| **Hosted RAG products** | Azure AI Search, Vertex AI Search, OpenAI Assistants File Search, AWS Bedrock Knowledge Bases |
| **Reranker** | Cohere Rerank, Voyage Rerank, Jina, BGE-reranker, ColBERT |
| **End-to-end** | Pinecone Assistant, Glean, Vectara, Perplexity-style |

## Companies / products using RAG

Nearly every production LLM product. Representative:

- **Anthropic Claude (file uploads)** ✅ — RAG over user docs.
- **OpenAI Assistants File Search, ChatGPT custom GPTs** ✅ — RAG primitives.
- **Perplexity, You.com, Phind** ✅ — RAG over web search results.
- **Notion AI, Slack AI, Google Workspace AI, Microsoft 365 Copilot** ⚠ — RAG over user content.
- **GitHub Copilot Chat (workspace), Cursor** ⚠ — RAG over codebase.
- **Glean, Vectara, Pinecone Assistant** ✅ — enterprise-search-as-RAG.
- **Most customer-support AI** (Intercom Fin, Zendesk AI, Ada) ✅ — RAG over KB.
- **AWS Bedrock Knowledge Bases, Azure AI Search, Vertex AI Search** ✅ — managed RAG services.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401) — Lewis et al. 2020 (original paper)
- [Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) — Anthropic Sept 2024
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [LangChain RAG tutorial](https://python.langchain.com/docs/tutorials/rag/)
- [LlamaIndex RAG patterns](https://docs.llamaindex.ai/)
- [What we learned from a year of building with LLMs (Part I)](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — RAG lessons section
- [Twelve RAG Pain Points and Solutions](https://towardsdatascience.com/12-rag-pain-points-and-proposed-solutions-43709939a28c/) — Wenqi Glantz
- [MTEB embeddings benchmark leaderboard](https://huggingface.co/spaces/mteb/leaderboard)

---

*Diagram source: [`../diagrams/src/basic-rag.d2`](../diagrams/src/basic-rag.d2)*
