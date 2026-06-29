# Inverted Index

**Aliases:** Postings List, Reverse Index, Search Index, Full-Text Index
**Category:** Data structure / Search
**Sources:**
[Brin & Page — *The Anatomy of a Large-Scale Hypertextual Web Search Engine* (1998)](http://infolab.stanford.edu/~backrub/google.html) ·
[Manning, Raghavan, Schütze — *Introduction to Information Retrieval* (Cambridge, 2008)](https://nlp.stanford.edu/IR-book/) ·
[Apache Lucene documentation](https://lucene.apache.org/core/) ·
[Elasticsearch documentation](https://www.elastic.co/guide/index.html) ·
[Postgres Full Text Search documentation](https://www.postgresql.org/docs/current/textsearch.html) ·
[ByteByteGo — Search Engine Architecture](https://blog.bytebytego.com/p/how-do-search-engines-work)

---

## Problem

> [!TIP]
> **ELI5.** You have a billion documents (web pages, products, log lines, emails). Someone types "fox jumps" — find the matching documents in milliseconds. You can't scan a billion documents per query. The trick is to **invert** the natural way data is stored: instead of "for each document, here are its words," store "for each word, here are the documents it appears in." Now finding "fox" is a single key lookup, and finding "fox AND jumps" is intersecting two lists. This **inverted index** is the data structure behind every search engine ever built — from Google and Bing down to Postgres's full-text search.

The forward representation of documents is natural but search-hostile:
```
doc1: "the quick brown fox"
doc2: "the lazy brown dog"
doc3: "quick fox jumps"
```

To find documents containing "fox," you must scan every document. For 1 billion documents, that's 1 billion string-search operations per query. Hopeless.

The inversion is the elegant fix:
```
"the"   → [doc1, doc2]
"quick" → [doc1, doc3]
"brown" → [doc1, doc2]
"fox"   → [doc1, doc3]
...
```

Now "fox" is a single key lookup returning a small list (doc1, doc3). "Quick AND fox" is the intersection of two lists. "Quick OR fox" is the union. "NOT fox" is the complement. All in time proportional to the *result size*, not the corpus size.

This is the single most important data structure in information retrieval. Lucene (the engine behind Elasticsearch, Solr, and many others), Postgres FTS, MongoDB text indexes, MySQL FULLTEXT, Algolia, Meilisearch, Tantivy, and every commercial search engine (Google, Bing, Baidu) are all variations on the inverted index.

Modern search engines add a great deal on top — relevance ranking (BM25, TF-IDF, learning-to-rank, neural rerankers), phrase queries, fuzzy matching, vector embeddings, faceting — but at the core, they all start with an inverted index.

## How it works

> [!TIP]
> **ELI5.** For each unique word ("term") in your corpus, keep a sorted list of which documents contain it (the "postings list"). To find documents matching a query: look up the terms, intersect or union their lists. To rank results: score each candidate using the term frequencies and other signals stored in the postings.

The inversion:

![Forward vs inverted index](../diagrams/svg/inverted-index.svg)

### The indexing pipeline

Building an inverted index from documents involves text-processing steps:

1. **Tokenize**: split text into terms. "Don't worry" → ["don", "t", "worry"]. Token boundaries are language-specific and harder than they look.
2. **Lowercase**: typically normalize case ("Fox" → "fox").
3. **Stopword removal**: optionally drop very common words ("the", "a", "of"). Saves index size at the cost of inability to search for them.
4. **Stemming or lemmatization**: reduce words to their root ("running" → "run"). Improves recall at the cost of precision. Porter stemmer is the classic English algorithm.
5. **Synonym expansion**: optionally add related terms.
6. **N-gram or shingle generation**: for phrase or fuzzy search, also index "quick brown" as a single term.
7. **Update postings**: for each term, append the document ID (and metadata) to its postings list.

Most search systems offer configurable analyzer pipelines per language (English vs Chinese vs Arabic have very different tokenization needs).

### Postings lists

A simple postings list is `[doc1, doc3, doc7, doc15, ...]`. Real postings lists store much more:

- **Document ID**: which document.
- **Term frequency (TF)**: how often the term appears in this document (for scoring).
- **Positions**: where in the document (for phrase queries — "quick brown" requires positions to be adjacent).
- **Offsets**: byte positions (for highlighting matches in the rendered result).
- **Payloads**: per-occurrence data (rare in practice).

The list is **sorted by doc ID**. Sorted lists enable fast set operations (intersection, union, difference) via merge-join: walk two lists in parallel, advance the smaller, emit when they match. Linear-time set operations in the size of the lists.

### Compression: why inverted indexes are small

Naively, postings lists could be huge. In practice they're tightly compressed (often 10-40% the size of the original text):

- **Delta encoding**: store gaps between doc IDs, not the IDs themselves. Doc IDs `[127, 128, 130, 132]` becomes `[127, 1, 2, 2]` — much smaller numbers.
- **Variable-byte encoding**: small numbers fit in 1 byte; larger in more. Big win on delta-encoded data.
- **Bitmap formats** (Roaring Bitmaps): for very dense postings (a term that's in nearly every doc), bitmaps win.
- **SIMD-friendly formats** (FOR, PFOR, SIMD-BP128): exploit CPU vector instructions for fast decode.
- **Block-based compression**: postings split into blocks; some blocks compressed differently from others.

Lucene's evolution over 20+ years has been heavily about better postings compression.

### Query processing

For a query "quick AND fox":
1. Look up postings for "quick" → `[doc1, doc3]`
2. Look up postings for "fox" → `[doc1, doc3]`
3. Intersect → `[doc1, doc3]`
4. Score each candidate (BM25, TF-IDF, etc.)
5. Sort by score, return top K.

For phrase query `"quick fox"`:
1. Get postings with positions.
2. For each candidate document containing both terms, check that "quick" position + 1 = "fox" position.

For prefix query `"qui*"`:
1. Look up all terms with prefix "qui" using a term dictionary (often a finite-state transducer for compact term-set storage).
2. Union all their postings.

For fuzzy queries, geo queries, range queries, etc., specialized index structures (in Lucene: BKD trees for geo/numeric ranges, FSTs for terms) layer on top.

### Relevance ranking

Returning matching documents is easy. **Returning the *right* matching documents** is the hard problem of information retrieval. Classic algorithms:

- **TF-IDF**: weight terms by frequency in document × rarity in corpus. The grandparent of search ranking.
- **BM25**: refined version with diminishing returns on term frequency and document length normalization. The default in Lucene/Elasticsearch since ~2009. Still hard to beat as a baseline.
- **PageRank / link-based ranking**: incorporate link structure. Google's original contribution.
- **Learning-to-rank**: machine-learned rankers using features (BM25 scores, click logs, freshness, authority). Standard for commercial search engines.
- **Neural rerankers**: BERT-based or LLM-based rerankers applied to the top-K from a classical ranker. Modern frontier.
- **Vector / embedding search**: encode documents and queries as dense vectors; retrieve by nearest neighbor. Increasingly hybridized with inverted indexes.

Most production search systems are now **hybrid**: a fast inverted-index first stage retrieves thousands of candidates; a learned-ranker or neural reranker re-orders the top hundreds.

### Distributed inverted indexes

For large corpora (billions of documents), the index is sharded:

- **Document partitioning**: each shard has its own complete index over its subset of documents. Queries fan out to all shards; results are merged. Standard for Elasticsearch, Solr.
- **Term partitioning**: postings for each term live on one shard. Queries with multiple terms fan out to multiple shards. Less common; more complex query processing but better for term-rare workloads.

Document partitioning dominates because it's simpler and parallelizes nicely. Replicas of each shard for read scaling and fault tolerance. Coordinator nodes route queries and aggregate.

### Modern additions: vector search

The 2020s saw the rise of **vector (embedding) search**: encoding documents and queries as dense vectors using deep learning models, then retrieving by approximate nearest neighbor (ANN — HNSW, IVF, ScaNN, Faiss).

This is **complementary** to inverted indexes, not a replacement:
- Inverted indexes excel at lexical / exact matches and rare-term queries.
- Vector indexes excel at semantic / similarity queries.
- Hybrid retrieval (combining BM25 + vector ANN) is now standard in RAG (retrieval-augmented generation) systems and modern search.

Elasticsearch, OpenSearch, Vespa, Weaviate, Qdrant, and others now support both inverted and vector indexes in the same engine.

### What's hard at scale

Building and serving inverted indexes for web-scale corpora has perennial hard problems:

- **Incremental updates**: re-indexing a billion documents from scratch is impossible. Lucene's segment-based architecture handles this — new docs go to new segments, segments are merged in the background. Reads see all segments.
- **Real-time freshness**: news search wants "indexed within seconds." Specialized in-memory or NRT (near-real-time) structures.
- **Deletes**: hard to remove a doc from an inverted index in place. Marked as deleted; reclaimed during segment merges.
- **Multi-language**: each language needs its own analyzer; query language detection is fuzzy.
- **Spam / SEO arms race**: not a data structure problem but a relevance problem.
- **Vocabulary size**: a billion-doc corpus has tens of millions of unique terms. Term dictionary structures (FSTs in Lucene) matter.

These are why search engines are mature, complex systems even after decades of engineering.

### A simple end-to-end picture

Imagine a small product search:

1. Product is added: `{id: 12345, name: "Brown Bear Plush Toy", description: "Soft brown plush bear ..."}`
2. Indexer tokenizes/stems: terms `brown`, `bear`, `plush`, `toy`, `soft`.
3. Postings update: `brown → [..., 12345]`, `bear → [..., 12345]`, etc.
4. User searches "brown bear toy":
5. Tokenize/stem query: `brown`, `bear`, `toy`.
6. Fetch postings, intersect.
7. Score each candidate (BM25 on each term).
8. Return top 10 sorted by score.

The same architecture, scaled up with sharding, compression, distributed merge, and learned ranking, is what powers Google.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Lucene segment-based** | Append-only segments; background merge. |
| **In-memory inverted index** | RAM-only; for hot data or real-time. |
| **Postings + positions** | Adds position info; enables phrase queries. |
| **Forward index (alongside)** | For result snippets and highlighting. |
| **Term dictionary (FST)** | Compact term-set storage. |
| **BKD tree** | Lucene's numeric / geo index. |
| **HNSW / IVF (vector index)** | Complementary; ANN over embeddings. |
| **Hybrid retrieval (BM25 + vector)** | Combine lexical + semantic. |
| **Distributed inverted index** | Document- or term-partitioned. |
| **Suffix tree / suffix array** | Alternative for substring search. |
| **n-gram index** | For fuzzy/typo-tolerant search. |

## When NOT to use

- **Small data set** — linear scan is fine for thousands of docs.
- **Pure structured queries** — relational queries don't need an inverted index.
- **Purely semantic similarity** — vector index alone may suffice (often still better hybrid).
- **When write amplification can't be tolerated** — index maintenance is significant.

---

## Real-world implementations

| Engine | Notes |
|---|---|
| **Apache Lucene** | The core library behind many search engines. |
| **Elasticsearch / OpenSearch** | Distributed Lucene; the dominant open-source search platform. |
| **Apache Solr** | Older, also Lucene-based. |
| **PostgreSQL FTS (`tsvector`)** | Native inverted-index-style full-text search. |
| **MySQL FULLTEXT** | Native FTS. |
| **MongoDB text index** | Native FTS. |
| **Algolia** | Hosted search SaaS; custom inverted-index engine. |
| **Meilisearch** | Open-source, easy-to-use search engine. |
| **Tantivy** | Rust Lucene equivalent; very fast. |
| **Vespa** | Search + ranking + vector engine (Yahoo!). |
| **Sphinx** | Older open-source search. |
| **Manticore** | Sphinx fork. |
| **Bleve** | Go inverted-index library. |
| **Whoosh** | Pure-Python search library. |
| **Apache Solr** | Lucene-based search platform. |
| **DuckDB FTS extension** | In-process FTS in OLAP. |
| **ClickHouse** | Recent inverted-index features. |
| **SQLite FTS5** | Inverted index in embedded SQLite. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Google** | The original (1998 paper); foundation of all web search. | ✅ Verified — [Brin & Page 1998](http://infolab.stanford.edu/~backrub/google.html) |
| **Microsoft Bing** | Inverted indexes at web scale. | ✅ Verified — Microsoft Research publications |
| **Amazon** | Product search; uses A9 (formerly Lucene-based, now ML-heavy). | ✅ Verified — Amazon A9 engineering posts |
| **Netflix** | Title search, content discovery. | ✅ Verified — Netflix Tech Blog |
| **Shopify** | Storefront search via Elasticsearch. | ✅ Verified — Shopify Engineering blog |
| **Wikipedia** | Search via Elasticsearch (CirrusSearch). | ✅ Verified — Wikimedia tech blog |
| **GitHub** | Code search and issue search via Elasticsearch (now custom Blackbird). | ✅ Verified — GitHub Engineering blog |
| **Twitter / X** | Tweet search via specialized inverted index (Earlybird). | ✅ Verified — Twitter Engineering blog, Earlybird paper |
| **Yelp, Etsy, eBay, Walmart, Target** | Product/listing search via Elasticsearch or similar. | ✅ Verified — engineering blogs |
| **Slack, Notion, Stripe, Atlassian** | Document/message search via Elasticsearch. | ✅ Verified — engineering blogs |
| **DataDog, Splunk, Sumo Logic** | Log search via inverted indexes (heavily customized). | ✅ Verified — products built on this |

---

## Further reading

- Brin & Page, *The Anatomy of a Large-Scale Hypertextual Web Search Engine* (WWW 1998) — Google's original paper.
- Manning, Raghavan & Schütze, *Introduction to Information Retrieval* (Cambridge 2008) — the IR textbook (free online).
- *Lucene in Action* (Hatcher & McCandless) — practical Lucene.
- *Elasticsearch: The Definitive Guide* — for the Elasticsearch ecosystem.
- *Search Engines: Information Retrieval in Practice* (Croft, Metzler, Strohman) — modern IR text.
- Twitter's Earlybird paper — real-time inverted index.
- Microsoft Research and Google Research papers on web-scale search.
- *Information Retrieval: Implementing and Evaluating Search Engines* (Büttcher, Clarke, Cormack) — implementation-focused.
- Postgres FTS documentation — the most accessible practical introduction.

---

*Diagram sources: [`../diagrams/src/inverted-index.d2`](../diagrams/src/inverted-index.d2).*
