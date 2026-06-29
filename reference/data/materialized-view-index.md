# Materialized View & Index Table

**Aliases:** Materialized View, Indexed View, Precomputed View, Secondary Index, Index Table, Local/Global Secondary Index
**Category:** Data / Read optimization
**Sources:**
[Microsoft Azure — Materialized View pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view) ·
[Microsoft Azure — Index Table pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/index-table) ·
[Oracle / PostgreSQL / SQL Server materialized view documentation](https://www.postgresql.org/docs/current/rules-materializedviews.html) ·
[DynamoDB documentation: Local and Global Secondary Indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html) ·
*Designing Data-Intensive Applications* (Kleppmann), Ch 3 and Ch 11

---

## Problem

> [!TIP]
> **ELI5.** Some queries are expensive (joins many tables, aggregates millions of rows, runs for seconds). If they run repeatedly and the underlying data changes slowly, you're doing the same expensive work over and over. Two related cures: **Materialized View** (compute the query *once*, store the result, serve it cheaply; refresh when data changes) and **Index Table** (build a separate sorted structure mapping query keys to data locations, so the query becomes a fast lookup instead of a scan). Both trade *write cost* (you must keep the precomputed structure in sync) for *read speed*. Both are foundational to making databases fast at scale.

Databases face a basic tension: **reads want pre-computed answers; writes want minimal work**. A few common pathologies:

- **Dashboards aggregating millions of rows on every page load.** "Total revenue by region, this quarter" against a 100M-row orders table. Run for every dashboard view. Same answer most of the time.
- **Filtering by non-primary-key columns.** `WHERE customer_id = 42` on a `orders` table keyed by `order_id`. Full table scan unless there's an index.
- **Expensive joins that produce simple-looking results.** "User profile with last 5 orders, total spend, loyalty status" joins 4 tables; the result is small.
- **Geo-spatial queries**, **full-text search**, **range queries on dates** — all benefit from specialized read structures.

Both materialized views and index tables solve a version of this problem:

- **Materialized View**: precomputes a *query result* (an aggregation, a join, a projection) and stores it as a named relation. Queries against the view are fast; the cost moves to keeping the view in sync with the source data.
- **Index Table** (a.k.a. secondary index): precomputes a *mapping* from non-primary-key columns to primary keys (or row pointers). Queries by indexed columns become fast lookups instead of scans; writes pay the cost of maintaining the index.

They're closely related — a secondary index is essentially a specialized materialized view that maps `(indexed_column → primary_key)`. Both pay the same fundamental cost: **write amplification** (every modification updates the precomputed structure too) in exchange for **read speedup**.

## How they work

> [!TIP]
> **ELI5.** Materialized view: "Here's the result of this expensive query. I saved it. Refresh it when the underlying data changes (now, on a schedule, or in the background)." Index table: "Here's a sorted list mapping `customer_id` to `order_id`. Look up `customer_id` to find the orders without scanning the whole table."

### Materialized View

A materialized view is a **stored, query-able relation** that holds the precomputed result of some query:

![Materialized View concept and refresh strategies](../diagrams/svg/materialized-view.svg)

```sql
CREATE MATERIALIZED VIEW revenue_by_region_quarter AS
SELECT region, quarter, SUM(amount) AS total_revenue
FROM orders
GROUP BY region, quarter;

-- Read from the view as if it's a table; instant.
SELECT * FROM revenue_by_region_quarter WHERE quarter = '2024Q1';
```

Without the view, every dashboard hit would re-execute the aggregation against the full orders table. With the view, dashboards hit precomputed rows.

The hard problem is **refresh**:

**1. Full refresh (rebuild).** Re-run the underlying query, replace the view's contents.
- Simple, correct, predictable.
- Doesn't scale to large views (long rebuild times).
- Stale between refresh runs.

**2. Incremental refresh.** Only update affected rows when source data changes.
- Much faster for small change sets.
- Complex to implement correctly (which view rows depend on which source rows?).
- Some databases support this only for certain query shapes (e.g., simple aggregates).

**3. On-demand.** Compute when queried, cache the result.
- First query is slow; subsequent fast.
- Effectively a cached result, not a true materialized view.
- Cache invalidation is the hard part.

**4. Event-driven.** Source-data changes emit events; a stream processor or job updates the view.
- Near-real-time without scheduled rebuilds.
- Requires event infrastructure ([CDC](cdc.md), [Outbox](outbox.md), Kafka, Flink).
- The basis of modern stream-processing systems (Kafka Streams, ksqlDB, Flink, Materialize).

Each strategy trades latency, freshness, complexity, and infrastructure cost.

### Where materialized views are built

Materialized views can live in many places:

- **In the same RDBMS**: PostgreSQL `MATERIALIZED VIEW`, Oracle, SQL Server `INDEXED VIEW`, MySQL via triggers. Cheap to set up; refresh is your problem.
- **In an analytics-tuned DB**: nightly ETL from OLTP to an OLAP store (Snowflake, BigQuery, Redshift, Druid, Pinot) where the precomputed answers live.
- **In Elasticsearch or another search engine**: index data shape optimized for read queries.
- **In a cache layer (Redis, Memcached)**: populated by application code or events.
- **In a stream processor (Kafka Streams, Flink, ksqlDB, Materialize)**: continuously-maintained materialized views from event streams.
- **As a CQRS read model**: a dedicated database tuned for a particular query pattern; see [CQRS](cqrs.md).

The choice depends on freshness requirements, infrastructure investment, and query patterns.

### Modern stream-processing view of the world

Sammelplatz database systems like **Materialize**, **ksqlDB**, **Flink SQL**, and **RisingWave** treat materialized views as first-class: you define a SQL view over event streams, and the engine maintains it incrementally in near-real-time. This is the modern reframe of the pattern: every read model is "just" a materialized view over a log of changes.

This view-as-stream-product duality (one of Kleppmann's main themes in DDIA) is influential: dashboards, search indexes, caches, derived tables, geo indexes — all are materialized views over the underlying event log.

### Index Table (Secondary Index)

An index table is a smaller, sorted structure that maps non-primary-key columns to primary keys (or row locators):

![Index Table: local vs global secondary index](../diagrams/svg/index-table.svg)

In an RDBMS:
```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
```

The DB builds and maintains a B-tree (or other index structure) keyed on `customer_id`. Queries like `WHERE customer_id = 42` now do an index lookup instead of a full table scan.

In NoSQL / distributed contexts, secondary indexes come in two flavors:

**Local Secondary Index (LSI)** — index lives with the partition:
- Each shard has its own LSI over its rows.
- Query by indexed column + partition key → hits one shard, fast.
- Query by indexed column alone → must fan out to all shards (scatter-gather).
- Cheap to maintain — no cross-shard writes.
- Used by DynamoDB LSI, Cassandra secondary indexes, MongoDB shard-local indexes.

**Global Secondary Index (GSI)** — index is its own partitioned table:
- Index is partitioned by the indexed column, not the base table's partition key.
- Query by indexed column → one shard lookup, no fan-out.
- Writes are now dual: write to base table + write to index table.
- Dual write is eventually consistent in most NoSQL systems (and that's a real semantic).
- Used by DynamoDB GSI, Cassandra Materialized Views, Spanner secondary indexes.

The trade-off is **read locality vs write cost**:
- LSI: cheaper writes, more expensive reads on indexed columns alone.
- GSI: cheaper reads, more expensive writes (and eventual consistency).

### Write amplification

Every secondary index (or materialized view) **multiplies write work**:

- Insert a row → write base table + update each index.
- Update a column → update base table + update each index that references that column.
- Delete a row → delete base + remove from each index.

For a heavily-indexed table, write amplification can be 5-10×. For [LSM-tree-based stores](lsm-tree.md), this stacks with the LSM's own write amplification, making secondary indexes especially costly. Some systems (Cassandra) discourage too many secondary indexes for this reason.

Rule of thumb: **index what you frequently filter, sort, or join by; avoid indexing everything.** Some workloads end up with index size larger than data size — sometimes that's right (analytical workloads), sometimes it's a smell.

### Consistency between base and view/index

When the materialized view or index is in a separate system (Elasticsearch index from a Postgres source, GSI on DynamoDB), you face an **eventual consistency** window:

- Write to base table succeeds.
- View / index updated asynchronously.
- Reads of the view during the lag see stale data.

Application code must:
- Tolerate the lag (most cases).
- Or fall back to the base table for fresh reads (often).
- Or use a strongly-consistent index if available (DynamoDB GSI is eventually consistent; some systems offer strongly-consistent options at higher cost).

For materialized views in the same RDBMS, refresh is typically explicit (manual `REFRESH MATERIALIZED VIEW`) — staleness is bounded by your refresh schedule.

### Compared to alternatives

- **Run the expensive query every time**: simple, always fresh, doesn't scale.
- **Cache the query result** ([cache-aside](../cache/cache-aside.md)): simpler than materialized view; less structured invalidation.
- **CQRS read model** ([CQRS](cqrs.md)): a fully-separate read database; the most general form.
- **Index-only queries** ("covering indexes"): an index that includes all the columns needed; query never touches the base table.
- **Denormalization in the schema**: pre-join data into wide rows at write time; a manual materialized view.

Materialized views and indexes are at different points on a spectrum from "compute on read" to "compute on write." Most production systems blend several approaches.

### A worked example: dashboard for an e-commerce site

- **Order list (most recent 50)**: primary-key descending scan on `orders` (cheap, no index/MV needed).
- **Customer history (orders for a customer)**: secondary index on `customer_id`. Local if querying within shard, global if not.
- **Total revenue by region this quarter**: materialized view aggregated by region+quarter. Refreshed nightly or via event-driven stream-processing.
- **Top 10 products by sales this week**: materialized view sorted by sales, top-K. Refreshed every few minutes.
- **Full-text search over product descriptions**: inverted index in Elasticsearch (see [Inverted Index](inverted-index.md)).

Different read patterns, different precomputed structures.

### When to use which

| Use case | Pattern |
|---|---|
| Filter by non-PK column | Secondary index |
| Sort by non-PK column | Secondary index |
| Joins or aggregations across many rows | Materialized view |
| Time-bucketed aggregations | Materialized view (often refreshed continuously) |
| Cross-system read (Postgres data in Elasticsearch) | Materialized view (event-driven) |
| Real-time analytics dashboard | Stream-processed materialized view (Flink, Materialize) |
| Free-text search | Inverted index (see [Inverted Index](inverted-index.md)) |
| Spatial queries | Spatial index (see [Range Partitioning & Geohash](range-partitioning-geohash.md)) |

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Materialized View (full refresh)** | Periodic rebuild. |
| **Materialized View (incremental)** | Only update affected rows. |
| **Materialized View (event-driven)** | Continuous updates from event stream. |
| **CQRS Read Model** ([page](cqrs.md)) | Dedicated read database; generalization of MV. |
| **Local Secondary Index (LSI)** | Index per partition; cheap writes. |
| **Global Secondary Index (GSI)** | Separately-partitioned index; eventual consistency. |
| **Covering Index** | Index includes queried columns; query avoids base table. |
| **Partial Index** | Index over a subset of rows (e.g., active only). |
| **Filtered Index (SQL Server)** | Same idea. |
| **Inverted Index** ([page](inverted-index.md)) | Specialized for full-text search. |
| **Index Table (Azure pattern)** | Microsoft's framing of secondary index. |
| **Denormalization** | Manual materialized view in schema design. |

## When NOT to use

- **Write-heavy workload with rarely-read aggregations** — write amplification dominates.
- **Tiny data set** — base-table scans are fine.
- **Highly dynamic schemas** — index/view design can't keep up.
- **When freshness must be strict** — eventually-consistent views may not be acceptable.
- **Too many indexes** — write amplification piles up; pick the ones that matter.

---

## Real-world implementations

| System | Notes |
|---|---|
| **PostgreSQL** | `MATERIALIZED VIEW`, `CREATE INDEX`, partial/expression indexes. |
| **Oracle** | Native materialized views, query rewrite. |
| **SQL Server** | Indexed views, filtered indexes. |
| **MySQL** | No native MV; emulated via tables + triggers. |
| **Snowflake / Redshift / BigQuery** | Materialized views as first-class. |
| **Druid / Pinot** | Pre-computed rollups (a form of materialized view). |
| **Cassandra** | Materialized Views (with caveats); secondary indexes. |
| **DynamoDB** | LSI and GSI as first-class. |
| **MongoDB** | Compound indexes, partial indexes, text indexes. |
| **Spanner** | Storing-clause secondary indexes. |
| **Kafka Streams / ksqlDB** | Continuously-maintained materialized views over streams. |
| **Apache Flink** | Stream-processed materialized views. |
| **Materialize, RisingWave** | Streaming SQL databases — materialized views are the product. |
| **Elasticsearch / OpenSearch** | Search indexes as materialized views from upstream sources. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Netflix** | Materialized views in Druid for real-time dashboards. | ✅ Verified — Netflix Tech Blog |
| **Uber** | Pinot for materialized views serving real-time analytics. | ✅ Verified — Uber Engineering blog |
| **LinkedIn** | Pinot (originated at LinkedIn); materialized rollups. | ✅ Verified — LinkedIn Engineering, Pinot origin story |
| **Airbnb** | Druid for materialized views. | ✅ Verified — Airbnb Engineering blog |
| **Twitter / X** | Pre-aggregated views for ranking and trending. | ⚠ Discussed in talks |
| **Amazon** | Materialized views and indexed tables underpin Retail systems. | ⚠ Standard practice; specifics vary |
| **Most analytical workloads everywhere** | Materialized views are foundational. | ✅ Industry standard |
| **DynamoDB customers** | LSI/GSI usage is ubiquitous. | ✅ AWS standard |
| **Postgres-heavy startups** | `MATERIALIZED VIEW` is standard for dashboards. | ✅ Common pattern |

---

## Further reading

- Microsoft Azure Architecture Center — Materialized View and Index Table patterns.
- PostgreSQL documentation on materialized views.
- DynamoDB developer guide — LSI and GSI sections.
- *Designing Data-Intensive Applications* (Kleppmann), Ch 3 and Ch 11.
- *Streaming Systems* (Akidau, Chernyak, Lax) — stream-processed materialized views.
- Materialize blog and documentation — modern stream-processing MV view.
- Druid / Pinot architecture documentation.
- Andy Pavlo's CMU database lectures — index internals.

---

*Diagram sources: [`../diagrams/src/materialized-view.d2`](../diagrams/src/materialized-view.d2), [`../diagrams/src/index-table.d2`](../diagrams/src/index-table.d2).*
