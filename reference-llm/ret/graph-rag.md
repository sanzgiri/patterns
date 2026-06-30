# GraphRAG (Knowledge Graph RAG)

**Aliases:** Microsoft GraphRAG, graph-augmented RAG, knowledge-graph RAG, LightRAG-style retrieval
**Category:** Retrieval
**Sources:**
[Edge et al. — From Local to Global: A Graph RAG Approach to Query-Focused Summarization (Apr 2024)](https://arxiv.org/abs/2404.16130) (Microsoft Research paper) ·
[Microsoft GraphRAG GitHub](https://github.com/microsoft/graphrag) (open-source implementation, 18k+ stars) ·
[LightRAG (HKU, 2024-2025)](https://github.com/HKUDS/LightRAG) (lighter-weight variant) ·
[Microsoft Research blog post](https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/)

---

## Problem

> [!TIP]
> **ELI5.** Basic RAG retrieves chunks by similarity. That works great for "what does the document say about X?" but breaks down on questions that need *connecting facts across the corpus*: "what are the main themes in this customer feedback?" (aggregation), "how is John Doe related to the merger?" (multi-hop relationship), "summarize the policy disagreements between teams" (synthesis across documents). **GraphRAG** is Microsoft's 2024 answer: at indexing time, use an LLM to *extract a knowledge graph* from the corpus (entities + relationships), cluster the graph into communities, generate per-community summaries at multiple levels of granularity. At query time: walk the graph for local questions, use pre-computed summaries for global questions. The cost is high (10-1000× basic RAG indexing) but the capability is genuinely new — answers to questions basic RAG can't touch.

The April 2024 Microsoft Research paper [From Local to Global: A Graph RAG Approach to Query-Focused Summarization](https://arxiv.org/abs/2404.16130) introduced GraphRAG and reported large improvements on global summarization queries vs. naïve vector RAG. The [open-source release](https://github.com/microsoft/graphrag) (July 2024) made it broadly usable. By 2025 the pattern has spawned a family: [LightRAG](https://github.com/HKUDS/LightRAG), Nano-GraphRAG, GraphRAG-Local, plus integrations in Neo4j, ArangoDB, and Microsoft Fabric.

The motivating use case in the paper: *podcast transcripts and news articles*. Ask "what are the main themes discussed across these 1000 podcast episodes?" Basic RAG retrieves the top-K most-similar chunks — but no individual chunk *is* "the main themes," so the answer is shallow. GraphRAG's community summaries are *built* to answer exactly this kind of holistic question.

## How it works

> [!TIP]
> **ELI5.** Two phases. **Indexing** (expensive, one-time): chop docs into chunks; for each chunk ask an LLM "what entities are here, and how are they related?"; merge the per-chunk extractions into a graph (people → relationships → orgs → events); run a community-detection algorithm (Leiden) to cluster the graph; LLM generates summaries for each cluster at multiple zoom levels (whole graph → sub-clusters → leaf entities). **Querying** (cheaper, per-request): for *local* questions, look up the entity in the graph and traverse to find relevant chunks. For *global* questions, fan out over the precomputed community summaries and map-reduce them. The graph + summaries are the new artifact — basic RAG never had this.

![GraphRAG](../diagrams/svg/graph-rag.svg)

### Indexing phase — building the graph

1. **Chunk documents.** Same as basic RAG.
2. **Per-chunk LLM extraction.** For each chunk, ask the LLM to extract: entities (name, type, description), relationships (source, target, description, strength), and claims (assertions with provenance). Output is structured (JSON).
3. **Graph assembly.** Merge per-chunk extractions into one graph: nodes = entities (deduplicated across chunks), edges = relationships (with weights and descriptions). Co-references resolved.
4. **Community detection.** Run Leiden algorithm (or Louvain) on the graph. Communities at multiple resolutions — small tight clusters → medium-sized groups → big themes.
5. **Per-community summary generation.** For each community at each level, LLM generates a summary of "what this community is about" using the entities, relationships, and original chunks within.
6. **Store everything.** Graph (Neo4j, parquet, custom), community structure, summaries (vector-embedded for retrieval), and original chunks (for citation).

This is *expensive*: an LLM call per chunk for extraction, plus per-community summarization, plus the community hierarchy can have hundreds of nodes. For Microsoft's paper-sized corpus (a few hundred news articles), it cost on the order of hundreds of dollars.

### Querying phase — two modes

**Local search** (entity-focused):
- "Who is the CEO of ACME Corp?"
- Identify entities in the query.
- Look them up in the graph.
- Traverse N-hop neighbors.
- Collect relevant chunks (those that contributed to nearby graph elements).
- Pass chunks + entity descriptions to LLM for answer.

Local search is essentially "vector RAG, but the graph helps you find the chunks." Useful when the question is specific and grounded in named entities.

**Global search** (theme/aggregation):
- "What are the main themes in this corpus?"
- Take the community summaries at the appropriate level.
- Map-reduce over them: each community produces a partial answer; aggregate.
- LLM synthesizes the partial answers into a final response.

Global search is where GraphRAG *uniquely* shines. The community summaries pre-encode the holistic understanding; query-time work is fast aggregation over them. Basic RAG simply cannot do this — there's no chunk that says "here are the themes."

### What questions GraphRAG handles well

- **Aggregation:** "What types of complaints appear most often?"
- **Multi-hop:** "Who funded the project that John Doe led, and what did the funder do next?"
- **Theme extraction:** "What are the recurring topics in customer feedback?"
- **Relationship mapping:** "How are these two organizations connected?"
- **Summarization at scale:** "Give me an overview of last quarter's product feedback."
- **Comparison across entities:** "How do Team A's concerns differ from Team B's?"

### What questions basic RAG handles better

- **Specific factual lookup:** "What is the warranty period for product X?" — single chunk answers it.
- **Quote retrieval:** "What did the report say verbatim about Y?" — vector similarity wins.
- **Cost-sensitive workflows:** "Just chat with my PDFs." — basic RAG is 100× cheaper.

The right call is often *both*: basic RAG for direct queries, GraphRAG for analytical / aggregation queries. Modern stacks route queries between them.

### Cost analysis — the elephant

| Phase | Basic RAG | GraphRAG |
|---|---|---|
| **Indexing** | Embed each chunk (~$0.0001/chunk) | LLM extraction per chunk + community summaries (~$0.01-0.10/chunk) |
| **Storage** | Vector DB | Vector DB + graph DB + summaries |
| **Query** | One LLM call after retrieval | Local: same. Global: K calls (map-reduce). |

For a corpus of 1M chunks:
- Basic RAG indexing: ~$100
- GraphRAG indexing: ~$10K-100K

This is *the* practical barrier to GraphRAG adoption. Production usage tends to cluster around:
- **High-value stable corpora** (legal case databases, internal company knowledge, research libraries) where the indexing cost amortizes over many queries.
- **Smaller corpora** (a few thousand chunks) where total cost is tens of dollars.
- **Subsidized R&D / intelligence applications** where the analytical capability justifies cost.

### The 2025 variants — making it cheaper

- **LightRAG** (HKU, late 2024): two-level graph (entity-level + theme-level), simpler community detection, dual-level retrieval. Much cheaper indexing, competitive quality on aggregation queries.
- **Nano-GraphRAG**: minimal reimplementation of Microsoft GraphRAG; same algorithm, less ceremony.
- **GraphRAG-Local**: run with local models (Llama, Gemma) for cost.
- **Modular GraphRAG**: separate entity extraction, graph construction, and summarization phases so you only run what you need.
- **Hybrid RAG + Graph**: use the graph only when query routing indicates it'll help, otherwise basic RAG.

### When to choose GraphRAG vs alternatives

| Need | Approach |
|---|---|
| Direct factual lookup | Basic RAG |
| Better retrieval quality on similar tasks | [Contextual retrieval](contextual-retrieval.md) + reranking |
| Multi-hop reasoning, occasional | Agentic RAG ([orchestrator-workers](../wf/orchestrator-workers.md) pattern with retrieval as a tool) |
| Heavy aggregation/synthesis | GraphRAG (or LightRAG) |
| Highest-quality analytical answers | GraphRAG with reranking layered |
| Cost-sensitive | Basic RAG + hybrid + rerank; only use Graph for premium tier |

### Engineering details that matter

- **Entity resolution.** Same entity referred to differently across chunks ("ACME Corp", "ACME Corporation", "ACME") must be unified. Cheap LLM call or fuzzy matching.
- **Schema for extraction.** A loose schema gives noisy graph; a tight one constrains LLM. Tune to corpus.
- **Community resolution levels.** Microsoft's implementation uses Leiden at multiple resolutions; pick levels based on query distribution.
- **Re-indexing strategy.** Graph is expensive to rebuild. Plan for incremental updates (new docs added, entities merged).
- **Graph query performance.** Large graphs need graph DBs (Neo4j, ArangoDB) for efficient traversal.
- **Summary refresh.** When new docs add to a community, summary should be regenerated or appended.
- **Provenance.** Every claim in a summary should trace back to original chunks. Microsoft's implementation includes this; some lightweight variants don't.

### Anti-patterns

- **GraphRAG for simple lookups.** Pay for capability you don't use.
- **No eval set.** GraphRAG quality is hard to measure subjectively; eval rigorously.
- **Cheap model for extraction.** Entity/relationship extraction quality drives downstream — invest in good extraction.
- **Skipping community detection.** Just having a graph isn't GraphRAG — the community structure + summaries are what enable global queries.
- **Treating it as a drop-in replacement.** It's a different capability, often used alongside basic RAG, not instead.
- **Ignoring re-indexing cost.** Plan for it; don't be surprised when adding 10% new docs requires re-running.

### Failure modes specific to GraphRAG

- **Entity drift over time.** Same person changes role; same company merges. Stale graph misrepresents reality.
- **Community ambiguity.** Leiden assigns each node to one community; in reality entities span multiple. Some variants allow overlap.
- **Summary hallucination.** LLM-generated summaries can confidently misstate. Provenance + verification critical.
- **Long-tail entities.** Rare entities mentioned once may not get extracted, can be silently absent.

### What's coming next (2025-2026)

- **Hybrid neuro-symbolic.** Combining LLM extraction with traditional NER for higher-precision graphs.
- **Streaming graph updates.** Incremental graph maintenance as docs change.
- **Cross-lingual graphs.** Single graph spanning multilingual corpora.
- **Multi-modal GraphRAG.** Including images, tables, code as nodes.
- **Agentic GraphRAG.** Agent dynamically traverses graph at query time rather than pre-computing summaries.

## Variants & related patterns

- [**Basic RAG**](basic-rag.md) — the alternative for direct queries.
- [**Advanced RAG**](advanced-rag.md) — the lighter alternative for query quality.
- [**Contextual retrieval**](contextual-retrieval.md) — often sufficient for non-aggregation queries.
- [**Reranking**](reranking.md) — combinable with GraphRAG.
- [**Orchestrator-workers**](../wf/orchestrator-workers.md) — agentic RAG can dispatch to GraphRAG or basic RAG.
- [**Routing**](../wf/routing.md) — route global queries to GraphRAG, local to basic.
- **Property Graphs** (Neo4j, ArangoDB) — the underlying graph DB pattern.
- **LightRAG** — cheaper variant.

## When NOT to use

- **Simple lookup-style queries** dominate your traffic.
- **Cost-sensitive deployments** where the indexing premium isn't justified.
- **Real-time evolving corpora** where rebuilding the graph isn't viable.
- **Tiny corpora** that fit in long-context window.
- **You don't have an eval set** for analytical-question quality.

## Implementations

| Tool | GraphRAG support |
|---|---|
| **Microsoft GraphRAG** | Canonical open-source ([github.com/microsoft/graphrag](https://github.com/microsoft/graphrag)) |
| **LightRAG** | Cheaper variant ([github.com/HKUDS/LightRAG](https://github.com/HKUDS/LightRAG)) |
| **Nano-GraphRAG** | Minimal reimplementation |
| **Neo4j** | Graph DB with LLM Graph Builder integration |
| **LlamaIndex** | `PropertyGraphIndex`, KnowledgeGraphIndex |
| **LangChain** | Graph chains, Neo4j integration |
| **Microsoft Fabric / Azure** | Hosted GraphRAG |
| **ArangoDB, Memgraph, Weaviate** | Hybrid graph + vector |
| **TigerGraph** | Graph DB with LLM integrations |

## Companies / products using GraphRAG

- **Microsoft** ✅ — published the paper, ships in Microsoft Fabric and Azure AI Search.
- **Microsoft 365 Copilot** ⚠ — uses graph-style retrieval over user content (Microsoft Graph).
- **Neo4j enterprise customers** ⚠ — financial services, pharma, intelligence.
- **Glean** ⚠ — uses graph augmentation in enterprise search.
- **Klarna, Bloomberg, Goldman Sachs (research)** ⚠ — known to use knowledge-graph-augmented retrieval.
- **Palantir** ⚠ — graph-based retrieval is core to their AI platform.
- **Tomoro AI, MultiOn** ⚠ — agentic systems with graph retrieval.

## Further reading

- [From Local to Global: A Graph RAG Approach](https://arxiv.org/abs/2404.16130) — Edge et al. April 2024 (canonical paper)
- [Microsoft GraphRAG GitHub](https://github.com/microsoft/graphrag) — open-source reference implementation
- [LightRAG paper](https://arxiv.org/abs/2410.05779) — Guo et al. Oct 2024 (cheaper variant)
- [GraphRAG blog](https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/) — Microsoft Research blog post
- [Knowledge Graphs and LLMs](https://neo4j.com/blog/knowledge-graph-vs-vectordb-for-retrieval-augmented-generation/) — Neo4j comparison post
- [LlamaIndex Property Graph guide](https://docs.llamaindex.ai/en/stable/module_guides/indexing/lpg_index_guide/)
- [Retrieval-Augmented Generation Survey](https://arxiv.org/abs/2312.10997) — broader RAG taxonomy

---

*Diagram source: [`../diagrams/src/graph-rag.d2`](../diagrams/src/graph-rag.d2)*
