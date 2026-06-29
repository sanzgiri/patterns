# Range Partitioning & Geohash

**Aliases:** Range Sharding, Range-based Partitioning, Geo Partitioning, Spatial Hashing
**Category:** Data distribution / Spatial
**Sources:**
[*Designing Data-Intensive Applications* (Kleppmann), Ch 6](https://dataintensive.net/) ·
[Google — Bigtable: A Distributed Storage System for Structured Data (OSDI 2006)](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/) ·
[Spanner: Google's Globally-Distributed Database (OSDI 2012)](https://research.google/pubs/spanner-googles-globally-distributed-database/) ·
[CockroachDB documentation: range-based partitioning](https://www.cockroachlabs.com/docs/stable/architecture/distribution-layer) ·
[Uber Engineering: H3 hexagonal hierarchical spatial index](https://www.uber.com/blog/h3/) ·
[Geohash Wikipedia](https://en.wikipedia.org/wiki/Geohash)

---

## Problem

> [!TIP]
> **ELI5.** [Consistent hashing](consistent-hashing.md) is great for point lookups ("give me user 42") because it spreads keys evenly across nodes. But it's terrible for **range queries** ("give me all orders from yesterday") because consecutive keys land on random nodes — you'd have to ask every node. **Range partitioning** solves that: it assigns each shard a *contiguous range* of keys, so range scans hit one or a few shards. The price: hotspots when keys are skewed (recent timestamps, sequential IDs). **Geohash** is the spatial-data version of the same idea: encode 2D coordinates as a 1D string where prefix length means geographic proximity, so "find restaurants near me" becomes a prefix scan.

[Consistent hashing](consistent-hashing.md) scatters consecutive keys (`order_1`, `order_2`, ..., `order_1000`) randomly across nodes. That's perfect for point lookups but terrible for queries like:

- "Give me the last 1000 orders."
- "Show me all events between 9am and 10am yesterday."
- "List all log entries for request ID prefix `abc123`."
- "Find restaurants within 1 km of (37.7749, -122.4194)."

With consistent hashing, every range query becomes a fan-out to *all* shards (a "scatter-gather"). That's expensive, slow, and scales badly.

**Range partitioning** trades the uniformity of hashing for **locality**: consecutive keys live on the same shard (or adjacent shards). Range scans are efficient because you touch one or two shards.

The trade-off is **hotspots**:

- If keys are timestamps and you keep inserting "now," 100% of writes hit one shard (the "latest" range).
- If keys are sequential IDs from an autoincrementing source, same problem.
- If access is skewed (some users way more active than others), their shards run hot.

The pattern is foundational to **wide-column stores** (HBase, Bigtable, Cassandra with `OrderPreservingPartitioner`), **NewSQL databases** (Spanner, CockroachDB, TiDB, YugabyteDB), and **spatial systems** (anything dealing with maps, GPS, geofencing).

**Geohash** is a clever specialization: it encodes 2D geographic coordinates as a 1D string where shared prefix length corresponds to geographic proximity. Once you have a geohash, you can range-partition and prefix-scan to find nearby items — turning spatial queries into ordinary range queries.

## How it works

> [!TIP]
> **ELI5.** Range partitioning: split the key space into contiguous ranges, one per shard. `user_id 1–1000` → shard 0, `user_id 1001–2000` → shard 1. Range query `user_id BETWEEN 500 AND 1500`? Hits shard 0 and shard 1. Geohash: encode (lat, lon) into a string where nearby points share a longer prefix. San Francisco = `9q8yy...`; a few blocks away = `9q8yyu...`. "Find points near here" becomes a prefix query.

### Range partitioning mechanics

The key space is divided into **contiguous ranges**:

```
Shard 0: keys [aa..ee]
Shard 1: keys [ee..jj]
Shard 2: keys [jj..pp]
Shard 3: keys [pp..zz]
```

A **shard map** (metadata service, coordinator) tracks the ranges. Lookups consult the map: "given key `koala`, find the shard that owns the range containing `koala`."

Common implementation choices:

- **Static ranges**: defined at table creation; doesn't auto-rebalance. Simple but inflexible.
- **Dynamic split/merge**: ranges automatically split when they grow too large; merge when they shrink. Used by Bigtable, HBase, Spanner, CockroachDB.
- **Two-tier**: an index of ranges (like a B-tree) makes lookups O(log N) instead of O(N).

Splits happen on size (e.g., split a range when it exceeds 512 MB), or on load (split a hot range to spread traffic), or on time (Spanner can split based on traffic patterns).

### Dynamic split/merge

The defining feature of production range-partitioned systems is **automatic split and merge**:

- A range grows large → split it into two ranges, each on a different node (or the same node for now).
- Adjacent small ranges → merge them to reduce overhead.
- A hot range → split it to spread load (even if size doesn't warrant a split).

This is how Bigtable, HBase, and Spanner avoid the worst of the hotspot problem without forcing operators to manually rebalance.

### The hotspot problem

Range partitioning's Achilles' heel is **skewed access**:

**Timestamp keys**: every new event has timestamp = "now"; every write hits the last range. The system is single-shard for writes regardless of cluster size. Solutions:
- **Reverse the timestamp** as a prefix: `0000_2024-01-15` vs `9999_2024-01-15` — distributes "now" across many ranges.
- **Salt the key**: prefix with a hash bucket: `bucket_5_2024-01-15`. Scatters writes; complicates range scans.
- **Switch to hash partitioning** for write-heavy timestamp data.

**Sequential IDs**: same pattern. Either use UUIDs (no sequential bias) or shard by hash of ID.

**Hot users**: a celebrity Twitter user gets way more reads than others. Their shard is hot. Solutions: split their shard further, replicate hot reads, use a cache layer.

### Compared to other strategies

The full partitioning landscape:

![Partitioning strategies side-by-side](../diagrams/svg/partitioning-strategies.svg)

| Strategy | Best for | Worst for |
|---|---|---|
| Modulo hashing | Stable N | Adding/removing nodes |
| [Consistent hashing](consistent-hashing.md) | Point lookups, frequent cluster changes | Range queries |
| Rendezvous (HRW) | Same as consistent + smaller routing tables | Same as consistent |
| Range partitioning | Range queries, locality | Sequential/timestamp keys |
| Geohash / H3 / S2 | Spatial proximity queries | Uniform load (cities are hotspots) |

Modern systems often combine: **hash + range** (DynamoDB's partition key + sort key; Cassandra's composite primary key). Hash to spread shards; range within a shard for ordered access.

### Geohash: spatial keys

Geohash encodes a (latitude, longitude) pair into a base32 string. The key property: **strings that share a longer prefix represent points that are geographically closer**.

![Geohash encoding levels and proximity queries](../diagrams/svg/geohash-encoding.svg)

Examples:
- `9q8yy` (5 chars) — covers a ~5km × 5km cell around San Francisco.
- `9q8yyk` (6 chars) — ~1.2km × 0.6km cell.
- `9q8yyk1` (7 chars) — ~150m × 150m cell.

Proximity query "find restaurants within 1 km of (37.7749, -122.4194)":

```sql
-- Compute query point's geohash at appropriate precision
-- e.g., '9q8yy' for ~5km cells
SELECT * FROM restaurants
WHERE geohash LIKE '9q8yy%'
ORDER BY ST_Distance(location, query_point)
LIMIT 100;
```

Plus checking 8 **neighbor cells** for boundary cases (your query point near the edge of a cell will have nearby points in the adjacent cell).

### Why geohash is great with range partitioning

If you range-partition by `geohash`, nearby locations live on the same shard. Spatial queries become local. This is the design behind:

- **Many ride-sharing systems** for driver/rider matching.
- **Restaurant finder apps** (Yelp, OpenTable).
- **Geo-targeted advertising**.
- **Geofencing services**.

### Modern alternatives to geohash

Geohash has well-known limitations: square cells don't tile uniformly, latitude distortion (cells get smaller toward poles), and discontinuities at the equator and prime meridian. Modern alternatives:

- **H3 (Uber)**: hexagonal cells; uniform neighbor distances; published by Uber for their dispatch system. Hexagons fit together better than squares for proximity queries.
- **S2 (Google)**: spherical projection; rich query primitives (cell, edge, polygon); used in Google Maps, MongoDB, and CockroachDB's spatial features.
- **Quadtree / R-tree**: classic 2D index structures used by PostGIS, traditional GIS systems.
- **KD-tree**: in-memory multi-dimensional partitioning.

Uber's H3 is particularly notable for ride-share use cases — its hexagonal cells make "find drivers within N rings of me" queries clean and uniform regardless of direction.

### Where range partitioning shines

Range partitioning is the right call when:

- **Range queries are common and important**: time-series, log search, ordered listings.
- **Spatial proximity matters**: geo queries with prefix encoding.
- **Locality matters**: keep "related" data together (all orders for one customer, all events for one session).
- **You're OK with operational complexity**: split/merge tooling, hotspot mitigation.

It's the wrong choice when:

- **Pure point-lookup workload**: hashing is simpler and faster.
- **Sequential keys without mitigation**: timestamp hotspots are real.
- **You can't tolerate hot shards**: hashing distributes more evenly.

### Compound: hash-then-range

The modern default in many distributed databases is **hash by part of the key, range within the partition**:

**DynamoDB**: partition key (hashed) + sort key (range within partition). Queries can range-scan within a partition but must point-lookup the partition.

**Cassandra**: composite primary key — `PRIMARY KEY ((partition_key), clustering_key1, clustering_key2)`. Partition key hashed; clustering keys ordered within the partition.

**Spanner / CockroachDB**: pure range partitioning with auto-split, but heavy hash-bucketing as a recommended schema pattern for write-heavy tables.

This pattern gives you the best of both: spread across shards (avoiding hotspots) while preserving ordering within each shard (enabling range scans on related data).

### Real-world hotspot war stories

- **Twitter** has famously struggled with celebrity user fan-out hotspots; the solution is a mix of caching, fan-out-on-read for celebrities, and sharded user-data layers.
- **Cassandra deployments** that used `OrderPreservingPartitioner` (range partitioning) by default famously developed hotspot issues; the default is now hash-based `Murmur3Partitioner`. Range partitioning is opt-in.
- **HBase** has documented anti-patterns around using sequential row keys; the recommendation is salting or reverse-timestamp prefixing.
- **Bigtable's "tablet" design** auto-splits ranges to handle growth and load — operators don't manually rebalance.

The lesson: range partitioning is powerful but requires schema design awareness.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Static range partitioning** | Fixed ranges; manual rebalance. |
| **Dynamic range partitioning** | Auto-split/merge (Bigtable, HBase, Spanner). |
| **Hash + range (composite)** | Hash to find partition, range within (DynamoDB, Cassandra). |
| **Geohash** | Spatial 2D → 1D encoding. |
| **H3 (Uber)** | Hexagonal spatial index. |
| **S2 (Google)** | Spherical spatial index. |
| **Quadtree / R-tree** | Spatial tree structures. |
| **Time-bucketed partitioning** | Special case for time-series. |
| **Directory-based partitioning** | Explicit mapping table; flexible. |
| **[Consistent hashing](consistent-hashing.md)** | The alternative for point lookups. |

## When NOT to use

- **Sequential / timestamp keys without salting** — guaranteed hotspot.
- **Pure point-lookup workload** — hashing simpler.
- **Tiny dataset** — partitioning at all may be premature.
- **When operators can't manage hotspots** — needs schema design discipline.

---

## Real-world implementations

| System | Notes |
|---|---|
| **HBase** | Range partitioning with auto-split. |
| **Google Bigtable** | The original tablet-based range partitioning. |
| **Google Spanner** | Range partitioning with auto-split, replicated via Paxos. |
| **CockroachDB** | Range partitioning with auto-split, replicated via Raft. |
| **TiDB / TiKV** | Range partitioning, Raft replication. |
| **YugabyteDB** | Hash or range; configurable. |
| **MongoDB** | Range or hash sharding (range historically). |
| **DynamoDB** | Hash partition + range sort key. |
| **Cassandra** | Hash-based by default; range available. |
| **PostGIS / Oracle Spatial** | Spatial indexing (R-tree). |
| **Elasticsearch** | Geohash and geo-grid aggregations. |
| **Uber H3** | Hexagonal spatial library. |
| **Google S2** | Spherical spatial library. |
| **Geohash libraries** | Available in every language. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Google** | Bigtable and Spanner use range partitioning at planetary scale. | ✅ Verified — [Bigtable](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/) and [Spanner](https://research.google/pubs/spanner-googles-globally-distributed-database/) papers |
| **Uber** | H3 spatial index for dispatch, ETA, surge zones. | ✅ Verified — [Uber H3 blog](https://www.uber.com/blog/h3/) |
| **Foursquare** | Geo-spatial partitioning for venue search. | ✅ Verified — Foursquare Engineering blog |
| **Lyft** | Geohash and H3-like spatial partitioning for matching. | ✅ Verified — Lyft Engineering blog |
| **Yelp** | Geohash-based proximity queries. | ⚠ Mentioned in talks |
| **Apple** | Cassandra at scale (mostly hash-partitioned but range examples exist). | ✅ Verified — Cassandra Summit talks |
| **Facebook / Meta** | HBase clusters for Messages used range partitioning. | ✅ Verified — Facebook HBase talks |
| **Pinterest** | HBase usage for various data. | ✅ Verified — Pinterest Engineering blog |
| **Snowflake / BigQuery** | Internal storage uses range-style partitioning for time-series tables. | ✅ Verified — public engineering posts |
| **CockroachDB customers** | Range partitioning everywhere CockroachDB is deployed. | ✅ Verified — CockroachDB customer stories |

---

## Further reading

- *Designing Data-Intensive Applications* (Kleppmann), Ch 6 — the clearest unified treatment.
- Bigtable paper (OSDI 2006) — the canonical reference for range partitioning at scale.
- Spanner paper (OSDI 2012) — adds Paxos replication to range partitioning.
- Uber's H3 blog and library docs — modern spatial indexing.
- Google S2 documentation — alternative spatial indexing.
- HBase reference guide — practical range-partitioning operations.
- CockroachDB architecture docs — range layer description.
- *Spatial Databases: With Application to GIS* (Rigaux et al.) — academic background.

---

*Diagram sources: [`../diagrams/src/partitioning-strategies.d2`](../diagrams/src/partitioning-strategies.d2), [`../diagrams/src/geohash-encoding.d2`](../diagrams/src/geohash-encoding.d2).*
