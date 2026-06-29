# Polyglot Persistence (and the Storage-Type Decision)

**Aliases:** Polyglot Persistence, Right Tool for the Job, Storage Taxonomy, Multi-Model Persistence
**Category:** Architecture / Data
**Sources:**
[Martin Fowler — *PolyglotPersistence* (2011)](https://martinfowler.com/bliki/PolyglotPersistence.html) ·
[Pramod Sadalage & Martin Fowler — *NoSQL Distilled* (2012)](https://martinfowler.com/books/nosql.html) ·
*Designing Data-Intensive Applications* (Kleppmann, 2017) — comparison of storage models ·
[Microsoft Azure — Data store classification](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/data-store-overview) ·
[Google Bigtable: A Distributed Storage System for Structured Data (OSDI 2006)](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/) ·
[Prometheus / InfluxDB / TimescaleDB documentation](https://prometheus.io/docs/)

---

## Problem

> [!TIP]
> **ELI5.** For decades the answer to "where do we put the data?" was "a relational database." That works for transactions, accounts, orders — but is awful for full-text search, time-series metrics, recommendation graphs, image storage, real-time leaderboards, or analytical aggregations on billions of rows. **Polyglot persistence** says: use the right tool for each workload. A typical modern application has a relational DB for orders, Redis for sessions, Elasticsearch for search, a time-series DB for metrics, S3 for files, and maybe a graph DB for recommendations — all behind one application. The cost is operational complexity; the benefit is using each store where it's strongest instead of where it's barely adequate.

The relational database is a remarkable invention. It handles a huge variety of workloads adequately. But "adequately" isn't always good enough:

- **Full-text search**: Postgres FTS works for small data; serious search needs Elasticsearch.
- **Time-series at scale**: a relational DB shard storing 1M metric points/second falls over; a TSDB does it on commodity hardware.
- **Graph traversals**: "find all users within 4 hops of Alice with similar purchase patterns" is awful in SQL, natural in Cypher.
- **Low-latency cache reads**: a Postgres query at 5ms; Redis at 200µs. 25× difference matters for hot paths.
- **Massive object storage**: storing 1 PB of images in Postgres is absurd; S3 costs cents per GB-month.
- **Analytical aggregations**: scanning 100M rows for a dashboard takes seconds in Postgres, milliseconds in BigQuery's columnar store.
- **Vector similarity**: nearest-neighbor search over learned embeddings — entirely new data type, native to vector DBs.

The pattern's name — **polyglot persistence** — was coined by Martin Fowler in 2011 (analogous to "polyglot programming" — using multiple languages for what each does best). The 2010s saw an explosion of specialized "NoSQL" stores; the 2020s see continued specialization (vector DBs, streaming SQL stores) plus consolidation around a smaller number of mature options.

The pattern is real and widely adopted, but it has a real cost: operational complexity, multi-store consistency, talent breadth. Modern advice: **don't reach for polyglot reflexively, but don't refuse it either** — use what the workload genuinely needs, and accept the cost when it pays off.

## How it works

> [!TIP]
> **ELI5.** Each storage technology was built around different assumptions about access patterns, consistency, durability, and scale. Pick a store whose assumptions match your workload's needs, not the other way around. Most non-trivial applications end up with 3-8 different stores, each owned by the team that uses it.

The pattern at the application layer:

![Polyglot persistence example](../diagrams/svg/polyglot-persistence.svg)

A typical e-commerce app might use:
- **Postgres** for orders, payments, customer accounts — ACID transactions essential.
- **MongoDB or DynamoDB** for product catalog — flexible schema per category, high read throughput.
- **Redis** for sessions, cart, rate limiting, leaderboards — sub-millisecond reads, ephemeral data.
- **Elasticsearch** for product search — full-text, faceting, relevance ranking.
- **Neo4j** for recommendation graph — "customers also bought," fraud rings.
- **InfluxDB / Prometheus** for application metrics — time-series compression and aggregation.
- **Snowflake / BigQuery** for analytics — columnar aggregation on billions of rows.
- **S3** for images, videos, logs — cheap, durable, infinitely scalable.

Each store handles what it's good at; none is asked to do something it's bad at.

### Trade-offs: when polyglot pays off and when it doesn't

**Benefits**:
- **Performance**: each workload runs on optimal infrastructure.
- **Cost**: a TSDB stores metrics 100× cheaper than the same data in Postgres.
- **Scale**: each store scales on its own axis (Redis horizontally for cache; Postgres vertically for OLTP; S3 infinitely for blobs).
- **Specialization**: rich features per workload (full-text relevance, graph algorithms, vector similarity).
- **Decoupling**: each service owns its store; teams move independently.

**Costs**:
- **Operational complexity**: 8 stores to monitor, back up, upgrade, patch.
- **Talent breadth**: team must know multiple technologies.
- **Cross-store transactions impossible**: need [Saga](../data/saga.md), [Outbox](../data/outbox.md), [CDC](../data/cdc.md).
- **Data synchronization**: keeping derived stores in sync with sources of truth.
- **Migration complexity**: schema changes ripple across multiple stores.
- **Debugging across boundaries**: a query that touches 4 stores has 4 places to look.

**A pragmatic rule**: start with Postgres. Add specialized stores only when you have a concrete workload where Postgres is genuinely failing — usually at one of three things: latency on hot reads (add Redis), full-text search (add Elasticsearch), or analytics (add a warehouse). Don't add stores speculatively.

### The storage type taxonomy

The major storage types and what they're each good at:

![Storage type taxonomy](../diagrams/svg/storage-taxonomy.svg)

A condensed reference:

**Relational (Postgres, MySQL, Oracle, SQL Server, Spanner, CockroachDB)** — ACID transactions, joins, mature ecosystem. The default for systems of record.

**Key-Value (Redis, Memcached, DynamoDB)** — O(1) lookups, very low latency, simple API. Caches, sessions, leaderboards.

**Document (MongoDB, Couchbase, DynamoDB doc, Cosmos)** — flexible schemas, nested data. Catalogs, content management, profiles.

**Wide-Column (Cassandra, HBase, ScyllaDB, Bigtable)** — massive write throughput, linear scale, time-series friendly. Telemetry, event logs, message timelines.

**Graph (Neo4j, JanusGraph, ArangoDB, Neptune)** — relationship traversals, paths, communities. Social networks, recommendations, fraud rings.

**Search (Elasticsearch, OpenSearch, Solr, Vespa, Algolia)** — inverted index, full-text, relevance, faceting. Product search, log search, autocomplete.

**Time-Series (Prometheus, InfluxDB, TimescaleDB, VictoriaMetrics, QuestDB)** — time-partitioned, compression, downsampling, retention. Metrics, monitoring, IoT.

**Columnar / OLAP (Snowflake, BigQuery, Redshift, Druid, Pinot, ClickHouse, DuckDB)** — massive scans, aggregations, MPP. BI, analytics, dashboards.

**Object (S3, GCS, Azure Blob, MinIO)** — cheap, durable, infinite scale. Images, videos, logs, data lake.

**Vector (Pinecone, Weaviate, Qdrant, Milvus, pgvector, Chroma)** — approximate nearest neighbor over embeddings. Semantic search, RAG, recommendation.

These categories blur — Postgres has JSON columns (document features), pgvector (vector features), FTS (search features). Some products straddle categories deliberately (ArangoDB is multi-model; DynamoDB is KV + document; Cosmos DB has five APIs). The categories are useful for thinking, not strict.

### Two important specializations: Time-Series and Wide-Column

Two storage types deserve closer attention because they're commonly misunderstood and frequently the right choice for specific workloads.

#### Time-Series Databases (TSDBs)

Time-series workloads have very specific access patterns:

![Time-series workload pattern](../diagrams/svg/timeseries-workload.svg)

The pattern:
- **Writes**: append-only, high QPS, monotonically-increasing timestamps, essentially no updates.
- **Reads**: by time range, grouped by tag/dimension, aggregated, almost never point lookups.
- **Retention**: raw for hours/days, downsampled for months/years, auto-delete oldest.

Specialized TSDBs (Prometheus, InfluxDB, TimescaleDB, VictoriaMetrics, M3) do things general DBs cannot:
- **Time-partitioned storage**: each chunk holds one time window; cheap deletion of expired chunks (drop the partition).
- **Heavy compression**: timestamps and values are highly compressible — delta-of-delta encoding, Gorilla algorithm (Facebook's compression, ~100× better than naive).
- **Downsampling rollups**: 5s resolution kept for 1 day; 1m for 30 days; 1h for 1 year — automatic and queryable.
- **Retention policies**: auto-drop chunks older than N days.
- **Cardinality awareness**: every tag combination creates a unique series; index by series ID, not row.
- **Time-aware query languages**: PromQL, InfluxQL, Flux, M3QL — designed for `rate()`, windowing, alignment.

For metrics, monitoring, IoT sensor data, financial market ticks — TSDBs are 10-100× more efficient than relational DBs at the same workload. **Always** use a TSDB for serious time-series workloads.

The downside: TSDBs are bad at non-time-series queries. Don't put your orders table in InfluxDB.

#### Wide-Column Stores

Wide-column stores (Bigtable, HBase, Cassandra, ScyllaDB) are a hybrid between row-oriented and column-oriented:

![Wide-column data model](../diagrams/svg/wide-column.svg)

The model: a sparse map `row_key → column_family → column_qualifier → value`. Rows can have millions of columns; different rows can have different columns; sparse columns cost nothing.

Why this matters:
- **Massive write throughput**: LSM-tree-based; Cassandra clusters routinely sustain 100K-1M writes/sec per node.
- **Linear horizontal scale**: just add nodes; consistent hashing distributes load.
- **Tunable consistency**: Cassandra lets you choose per query (ONE, QUORUM, ALL).
- **Time-series friendly**: wide rows with timestamp columns are natural — message timelines, IoT telemetry, clickstreams.
- **Efficient range queries within a row**: clustering columns enable ordered scans.
- **Sparse storage**: missing columns cost nothing; great for irregular data.

Wide-column stores power some of the world's largest systems: Facebook Messages, Discord chat, Apple iCloud, Netflix viewing history. **For very-high-write, scale-out, time-bucketed workloads, they're often the right choice.**

The downside: query patterns must match the schema. Secondary indexes are weak. Cross-row transactions are limited. The team must understand the access pattern upfront.

### Real-world combinations

Some classic combinations seen in industry:

| Company | Storage stack (approximate) |
|---|---|
| **Netflix** | Cassandra (viewing history, account), EVCache (Memcached), Elasticsearch (search), S3 (videos), Redshift/Druid (analytics), Memcached |
| **Twitter** | MySQL/Vitess (tweets historically), Manhattan KV, Cassandra, Elasticsearch (search), Redis (timelines) |
| **Uber** | MySQL/Schemaless (driver/rider data), Cassandra (geo), Elasticsearch (search), Pinot (analytics), Hive/S3 (data lake) |
| **LinkedIn** | MySQL/Espresso (member data), Cassandra, Pinot (analytics), Voldemort (KV), Elasticsearch |
| **Discord** | Cassandra/ScyllaDB (messages), Redis, Elasticsearch, S3 |
| **Airbnb** | MySQL (transactional), Druid (analytics), HBase, Elasticsearch, S3 |
| **Shopify** | MySQL (everything OLTP — they're famously a "boring tech" shop), Elasticsearch, BigQuery |
| **Etsy** | MySQL, Solr, Memcached, BigQuery |
| **Stripe** | MongoDB historically; PostgreSQL with custom infrastructure; significant data warehousing |

Note the variance: some companies (Shopify, Stripe) stay closer to a single relational core and use specialized stores narrowly. Others (Netflix, Twitter, Uber) embrace many specialized stores. Both approaches work depending on the engineering culture and workload.

### Keeping polyglot data consistent

When source-of-truth lives in one store but is also kept in another (Postgres orders also indexed in Elasticsearch for search), you need a consistency strategy:

- **[Change Data Capture (CDC)](../data/cdc.md)**: stream the DB's binlog/WAL to downstream stores. The most robust general-purpose approach.
- **[Transactional Outbox](../data/outbox.md)**: write to DB + outbox in the same transaction; a relay reads outbox and publishes events.
- **Dual-write**: application writes to both stores. Fragile — partial failures lead to drift.
- **Periodic full sync**: nightly ETL of source → derived. Coarse but reliable.
- **[Event sourcing](../data/cqrs.md)**: all data starts as events; both source and derived stores are projections. Most architecturally pure but invasive.

For most teams: CDC (via Debezium, Kafka Connect, AWS DMS) is the modern default. Easier than outbox to add to existing apps; more reliable than dual-write.

### Modern trends

A few notable trends in the 2020s:

- **Vector databases** as a new category, driven by RAG and embedding-based search.
- **Hybrid stores**: pgvector adds vector to Postgres; Elasticsearch adds vector to inverted index; many existing stores grow.
- **Multi-model**: ArangoDB, Cosmos DB, Couchbase — one product, multiple APIs.
- **Streaming SQL**: Materialize, RisingWave, ksqlDB — materialized views over event streams as a first-class store.
- **Data lakehouses** (Delta Lake, Iceberg, Hudi): merge data warehouse and data lake; transactional storage on object stores.
- **Embedded analytical stores** (DuckDB, ClickHouse local) — analytics without a server.

The space is not consolidating; if anything, it's expanding. Polyglot persistence as a practice is more relevant than ever.

### A pragmatic decision framework

When facing "what store should we use for X?":

1. **What's the access pattern?** Reads vs writes; point vs range vs scan; latency vs throughput.
2. **What's the consistency requirement?** ACID, strong, eventual, none.
3. **What's the scale?** Bytes? Megabytes? Petabytes?
4. **What's the team's experience?** Don't add a graph DB if no one knows graph databases.
5. **What's the ops cost?** Self-hosted? Managed? Multi-cloud?
6. **What's the cost?** Storage cost, compute cost, license cost.
7. **What's the migration cost** if we get this wrong?

Often the answer is "Postgres can do this for now; specialize when it's failing." Sometimes the answer is "this needs Elasticsearch from day one." Both are valid — depend on the specifics.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Single-store** | Use one DB for everything; simple but limited. |
| **Polyglot persistence (classic)** | Multiple specialized stores. |
| **Multi-model DB** | One product, multiple APIs (Cosmos, ArangoDB). |
| **Database-per-service** ([page](../data/database-per-service.md)) | Each microservice has its own store; often heterogeneous. |
| **Lambda architecture** | Batch + speed layers; effectively polyglot. |
| **Kappa architecture** | Stream-only; one store, often. |
| **Data lakehouse** | OLAP + transactional on object storage. |
| **Materialized view fanout** | Sources in one store, projections in many. |
| **Time-series specialization** | TSDB for metrics. |
| **Wide-column specialization** | Cassandra/HBase for big-write workloads. |
| **CDC-based propagation** | Source store + downstream sinks via CDC. |

## When NOT to use

- **Small team / small system** — operational cost too high.
- **No clear specialized workload** — single store suffices.
- **Without observability** to monitor multiple stores.
- **Without consistency strategy** — drift will accumulate.
- **As technology fashion** — pick stores you genuinely need, not "to be modern."

---

## Real-world implementations

| Store type | Common picks |
|---|---|
| Relational | Postgres, MySQL, Oracle, SQL Server, Aurora, Spanner, CockroachDB |
| Key-Value | Redis, Memcached, DynamoDB |
| Document | MongoDB, Couchbase, DynamoDB, Cosmos DB |
| Wide-Column | Cassandra, HBase, ScyllaDB, Bigtable |
| Graph | Neo4j, JanusGraph, ArangoDB, Neptune, TigerGraph |
| Search | Elasticsearch, OpenSearch, Solr, Vespa, Algolia, Meilisearch |
| Time-Series | Prometheus, InfluxDB, TimescaleDB, VictoriaMetrics, M3, QuestDB |
| OLAP / Columnar | Snowflake, BigQuery, Redshift, Druid, Pinot, ClickHouse, DuckDB |
| Object | S3, GCS, Azure Blob, MinIO |
| Vector | Pinecone, Weaviate, Qdrant, Milvus, pgvector, Chroma |
| Lakehouse | Delta Lake, Iceberg, Hudi |
| Stream-SQL | Materialize, RisingWave, ksqlDB, Flink SQL |

## Companies / canonical uses

| Where | Polyglot stack | Status |
|---|---|---|
| **Netflix** | Cassandra + EVCache + Elasticsearch + S3 + Redshift/Druid. | ✅ Verified — Netflix Tech Blog |
| **Twitter / X** | MySQL/Vitess + Manhattan + Cassandra + Elasticsearch + Redis. | ✅ Verified — Twitter Engineering blog |
| **Uber** | MySQL/Schemaless + Cassandra + Elasticsearch + Pinot + Hive/S3. | ✅ Verified — Uber Engineering blog |
| **LinkedIn** | MySQL/Espresso + Cassandra + Pinot + Voldemort + Elasticsearch. | ✅ Verified — LinkedIn Engineering blog |
| **Discord** | Cassandra → ScyllaDB + Redis + Elasticsearch + S3. | ✅ Verified — Discord Engineering blog |
| **Airbnb** | MySQL + Druid + HBase + Elasticsearch + S3. | ✅ Verified — Airbnb Engineering blog |
| **Shopify** | MySQL-dominant + Elasticsearch + BigQuery (less polyglot by choice). | ✅ Verified — Shopify Engineering blog |
| **Etsy** | MySQL + Solr + Memcached + BigQuery. | ✅ Verified — Etsy Engineering blog |
| **Stripe** | MongoDB historically; complex DW. | ✅ Verified — Stripe Engineering blog |
| **Most "modern" microservices orgs** | Polyglot is the de facto pattern. | ✅ Industry-wide trend |

---

## Further reading

- Martin Fowler, *PolyglotPersistence* (2011) — the bliki post that named it.
- Sadalage & Fowler, *NoSQL Distilled* (2012) — accessible NoSQL guide.
- *Designing Data-Intensive Applications* (Kleppmann, 2017) — the modern reference for thinking about storage choices.
- Microsoft Azure Architecture Center — data store classification.
- Bigtable paper (OSDI 2006), Dynamo paper (SOSP 2007), Cassandra paper (LADIS 2009).
- Pavlo's CMU database courses — comprehensive architecture comparisons.
- *Time Series Databases* (Dunning & Friedman) — practical TSDB guide.
- DDIA's chapters on different storage engines (Ch 3).
- Most major engineering blogs (Netflix, Uber, LinkedIn, Discord) — specific case studies of polyglot at scale.

---

*Diagram sources: [`../diagrams/src/polyglot-persistence.d2`](../diagrams/src/polyglot-persistence.d2), [`../diagrams/src/storage-taxonomy.d2`](../diagrams/src/storage-taxonomy.d2), [`../diagrams/src/timeseries-workload.d2`](../diagrams/src/timeseries-workload.d2), [`../diagrams/src/wide-column.d2`](../diagrams/src/wide-column.d2).*
