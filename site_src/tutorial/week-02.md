# Week 2 — Storage, Indexes, Caches, and Sketches

!!! abstract "Learning goal"
    Match a workload to a storage structure, place caches and materialized views safely, and distinguish probabilistic membership, counting, cardinality, and similarity.

## Lesson: choose by access pattern

| Need | Structure | Main trade-off |
|---|---|---|
| Point lookup and ordered updates | B-tree | Random-page updates, strong range behavior |
| High write throughput | LSM-tree | Compaction and read amplification |
| Durable acknowledged writes | WAL | Extra write and recovery work |
| Consistent concurrent reads | MVCC | Version cleanup and long snapshots |
| Repeated hot reads | Cache | Staleness and invalidation |
| Expensive known query | Materialized view | Rebuild and freshness cost |
| Term-to-document search | Inverted index | Index maintenance and ranking |

The choice is driven by ordering, mutation rate, scans, durability, and query shape—not by a generic “SQL versus NoSQL” preference.

### Approximation toolbox

| Question | Structure | Error behavior |
|---|---|---|
| Could this key exist? | Bloom filter | False positives, no false negatives |
| How many distinct keys? | HyperLogLog | Approximate cardinality |
| How frequent is key X? | Count-Min Sketch | Overestimates frequency |
| How similar are two sets? | MinHash | Approximate Jaccard similarity |

Bloom filters are commonly attached to LSM-tree files to avoid disk reads for definite misses. MinHash is useful for near-duplicate documents or product descriptions; it measures shared set elements, not semantic meaning.

## Worked example: e-commerce search

```text
catalog database ──> CDC/indexer ──> inverted search index
        │                         └─> materialized product view
        └─────────────────────────> cache invalidation

search request ──> query service ──> search index
detail request ──> cache-aside ──> product view/database
```

The catalog database is authoritative. Search indexes and product views are derived and rebuildable. Product availability and price need stricter freshness than descriptive text.

During a sale, use cache hierarchy, hot-key protection, and request coalescing. For duplicate descriptions, use MinHash or SimHash before indexing, followed by exact or human verification where necessary.

## Practice: now answer

1. Why does an LSM-tree benefit from a Bloom filter?
2. Which structure estimates unique visitors?
3. Which structure finds unusually frequent IP addresses?
4. Why is MinHash not a replacement for embedding search?
5. What happens when a materialized product view is stale or corrupted?

## Interview drill

Design product search for a marketplace. Explain the source of truth, indexing pipeline, cache policy, freshness contract, rebuild process, and behavior when the search index is unavailable.

!!! success "Self-check"
    You should be able to say “Bloom = membership, HLL = distinct count, Count-Min = frequency, MinHash = set similarity” without conflating their guarantees.
