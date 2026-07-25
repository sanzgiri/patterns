# Week 2 — Storage, indexes, caches, and probabilistic sketches

## Outcome

By the end of this week, you should be able to match a workload to a storage structure, explain the read/write trade-offs of B-trees and LSM-trees, place caches and materialized views safely, and distinguish probabilistic membership, counting, cardinality, and similarity tools.

## Patterns and material

- [B-Tree](../../reference/data/btree.md)
- [LSM-Tree](../../reference/data/lsm-tree.md)
- [WAL](../../reference/data/wal.md)
- [MVCC](../../reference/data/mvcc.md)
- [Probabilistic Sketches](../../reference/data/probabilistic-sketches.md)
- [Cache-Aside](../../reference/cache/cache-aside.md)
- [Cache Hierarchy](../../reference/cache/cache-hierarchy-storage-tiering.md)
- [Materialized View](../../reference/data/materialized-view-index.md)
- [Inverted Index](../../reference/data/inverted-index.md)

## Session 1 — Choose by access pattern

Ask what the system does most often:

| Workload | Useful structure | Why |
|---|---|---|
| Point lookup and updates | B-tree or hash index | Predictable lookup and update path |
| High write throughput and immutable batches | LSM-tree | Sequential writes, later compaction |
| Ordered range scans | B-tree or sorted files | Key order is useful |
| Full-text term lookup | Inverted index | Term → document postings |
| Repeated hot reads | Cache | Avoid repeated lower-tier work |
| Predefined expensive query | Materialized view | Compute once, read cheaply |

The choice is not “SQL versus NoSQL.” It is a choice about ordering, mutation, scan behavior, durability, and operational cost.

## Session 2 — Durability and concurrency

A typical durable write path is:

```text
request → WAL append → memory structure → acknowledgement
                           ↓
                    background flush/compaction
```

The WAL protects acknowledged intent before the main structure is persisted. MVCC gives readers a coherent version while writers proceed, but it creates version cleanup and snapshot-lifetime concerns.

Compare:

- B-tree: in-place page updates, good reads and range scans, random-write pressure;
- LSM-tree: sequential writes and high write throughput, compaction/write amplification and read amplification;
- cache: fast but not authoritative unless deliberately designed as such;
- materialized view: fast query path but asynchronous staleness and rebuild complexity.

## Session 3 — The four approximation questions

Do not call every small-memory structure a “Bloom filter.” Ask the question:

| Question | Structure | Guarantee |
|---|---|---|
| Could this key exist? | Bloom filter | No false negatives; possible false positives |
| How many distinct keys? | HyperLogLog | Approximate cardinality |
| How frequent is key X? | Count-Min Sketch | Approximate over-count |
| How similar are two sets? | MinHash | Approximate Jaccard similarity |

Bloom filters are especially useful in LSM-tree SSTables: a negative answer avoids disk I/O. MinHash is useful for near-duplicate documents or product descriptions. It is not a semantic similarity method; use embeddings when meaning matters more than shared tokens.

## Session 4 — Case: e-commerce search

### Prompt

“Design product search and a product-detail page for a large marketplace.”

### Architecture

```text
catalog writes → source database → CDC/indexer → inverted search index
       │                              └──────→ materialized product view
       └─────────────────────────────────────→ cache invalidation/events
search request → query service → search index → ranked product IDs
product request → cache-aside → materialized view/source database
```

The source database owns product truth. The search index and materialized view are derived and rebuildable. Product detail can tolerate brief staleness, but price and availability need a stronger freshness policy; do not hide those behind an unbounded cache TTL.

### Requirement changes

1. Traffic spikes during a sale: add cache hierarchy, hot-key protection, and request coalescing.
2. Duplicate descriptions flood the catalog: use MinHash/SimHash pre-indexing, then human or exact verification.
3. Analytics needs unique visitors and top search terms: use HLL and Count-Min Sketch, with a batch reconciliation path for critical reports.

## Deliverable and rubric

Submit a workload-to-structure table, a write/read diagram, cache invalidation policy, rebuild plan, and a one-paragraph explanation of Bloom filter versus MinHash.

Score 0–3 on: access-pattern matching, durability, staleness reasoning, approximation semantics, cache failure handling, and ability to explain rebuilds.

