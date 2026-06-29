# Segmented Log

**Aliases:** Log Segmentation, Rolling Log, Log File Rotation, Partitioned Log
**Category:** Data / Building block
**Sources:**
[Joshi — Patterns of Distributed Systems](https://martinfowler.com/articles/patterns-of-distributed-systems/) ·
Kleppmann *DDIA*, Ch 3 ·
[Kafka log design docs](https://kafka.apache.org/documentation/#log)

---

## Problem

> [!TIP]
> **ELI5.** You've been writing in one giant notebook for years and it's now 10,000 pages thick. You can never throw any pages away (it's a single file). Finding a specific entry takes forever. The book is also too heavy to carry to backup. Better: switch to a new notebook every month. Old notebooks can be archived or trashed; current writing stays small and fast.

A [WAL](wal.md) or [replicated log](replicated-log.md) grows monotonically. As a **single file**, this creates several problems that get worse with time:

- **Truncation is impossible.** You cannot delete the first half of a file without rewriting the whole thing. So old, no-longer-needed entries pile up forever, consuming disk indefinitely.
- **Lookups slow down.** Finding a specific log position requires indexing into an ever-growing file. Index structures grow without bound.
- **Backups become painful.** Backing up a 1 TB ever-growing file is fundamentally different (and worse) than backing up many sealed 1 GB files.
- **Recovery time scales with size.** A crash means scanning back to the last checkpoint; if the file is huge, this is slow.
- **Compaction becomes a "rewrite the world" operation** instead of a per-segment operation.

You need a log that grows but stays manageable — old portions can be discarded cheaply, lookups remain fast, backups are incremental, and compaction is local rather than global.

## How it works

> [!TIP]
> **ELI5.** Don't write one ever-growing file. Roll over to a new file every gigabyte (or every hour). The current file is **active** (being appended); older files are **sealed** (read-only). When old files are no longer needed — `unlink()` them. Reads use a small lookup table to find which file contains the desired offset.

A segmented log splits the conceptually-unbounded log into a sequence of fixed-size **segment files**. Exactly one segment is **active** at any time — the one being appended to. When the active segment hits a configured threshold (size, age, or record count), it is **sealed** (closed, marked read-only) and a new active segment is opened.

![Segmented log: structure and retention](../diagrams/svg/segmented-log-structure.svg)

The active segment receives writes. Sealed segments are immutable — they can be read concurrently by many consumers, served via zero-copy (`sendfile(2)` on Linux), compressed in place, and copied for backup. The oldest sealed segments, once they fall below the **retention horizon** (size limit, age limit, or [LWM](../block/hwm-lwm.md)), become **truncation candidates** and can be deleted by a simple `unlink()` of the whole file — free, no compaction needed.

Alongside each segment, an **index file** maps logical offsets to byte positions within the segment. Indexes are kept **sparse** — one entry per N records rather than one per record — to keep their size negligible while still bounding lookup time.

### Read path

The read path is where the segmented design pays its biggest dividend:

![Segmented log read path](../diagrams/svg/segmented-log-read.svg)

To find a record at logical offset 7500, the system:

1. **Picks the segment**: binary-search the list of segment base offsets. With sorted base offsets `[1, 2001, 4001, 6001, 8001, ...]`, the search finds segment 040 (base 6001 ≤ 7500 < base of next segment 8001). This is O(log n) in the number of segments — and there are rarely more than a few hundred even for terabyte-scale logs.
2. **Uses the sparse index**: open the segment's `.index` file, binary-search for the nearest indexed offset ≤ 7500. Index entries are spaced every few KB, so the nearest is at most a few records away.
3. **Seeks and scans**: `seek()` into the data file at the indexed byte position, scan forward a few records to find exact offset 7500.
4. **Reads with zero-copy**: for streaming reads (Kafka consumers), the server uses `sendfile()` to push bytes directly from the page cache to the network socket, never copying through user space.

The combined access cost is O(log #segments + log #index_entries + small_scan) — bounded and predictable even as the log grows to terabytes.

### Rotation and retention policies

A segmented log has two orthogonal sets of policies:

- **Rotation** decides *when* to seal the active segment and start a new one. Common triggers: size threshold (Kafka's `log.segment.bytes`, default 1 GB); age threshold (`log.roll.ms`, default 7 days); record count.
- **Retention** decides *which* sealed segments to keep. Options: total size cap (`log.retention.bytes`); time-based (`log.retention.hours`, default 7 days); commit-based (Raft keeps everything ≥ LWM, deletes everything below); compaction (per-key retention — Kafka's compacted topics retain only the latest value per key).

These are independent — you can rotate every hour but retain for 30 days. The two-axis design covers most production needs without complex tuning.

### Compaction variants

For systems with idempotent or replaceable records, **log compaction** is a more sophisticated retention strategy. Rather than deleting whole segments by age, the compactor periodically rewrites segments: for each key, only the most recent record is kept; older records for the same key are dropped. This bounds storage by the number of distinct keys rather than the volume of writes.

Kafka compacted topics use this to back KV stores. RocksDB's LSM-tree merges combine sorted segments into larger ones, dropping superseded records along the way. **Sorted String Table (SSTable)** based engines treat each segment as an immutable sorted file and compact pairs of SSTables into single larger ones, merge-sort style.

### Family of segmented logs

The pattern is everywhere once you recognize it. **Postgres `pg_wal/`** rotates WAL files at 16 MB each (the segment size on disk). **MySQL binlog** rotates per file (`max_binlog_size`). **Cassandra commitlog** rotates by size. **Kafka** is the most prominent and rigorous user — log segments are first-class, exposed in `log.dirs` per topic-partition. **RocksDB/LevelDB** SSTables are segments. **Apache BookKeeper ledgers** are append-only segments. **etcd WAL** files are segmented. **HDFS edit logs** are segmented. The pattern is universal because the constraints — append performance, bounded recovery time, cheap retention — are universal.

A practical operational note: at very large scale (Kafka brokers with tens of thousands of partitions), the number of files becomes the operational concern. Each partition has its active segment plus several sealed ones plus index files — multiplied by partitions, the file count can hit hundreds of thousands. This stresses inode caches, directory listing performance, and backup tooling. The mitigations include segment size tuning, per-broker partition limits, and tiered storage (moving cold segments to object storage; Kafka KIP-405).

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Size-based rotation** | Roll at N MB. Kafka default. |
| **Time-based rotation** | Roll every N hours. Common for log files. |
| **Compacted log / Log compaction** | Per-key retention rather than time-based. Kafka compacted topics. |
| **Tiered storage** | Move cold sealed segments to cheap object storage. Kafka KIP-405, Pulsar tiered storage. |
| **SSTable (Sorted String Table)** | Sealed segment that's internally sorted by key — enables merge-based compaction (LSM-trees). |
| **Sparse index** | Co-located index file with one entry per N records, balancing lookup speed vs index size. |
| **Memory-mapped segments** | Active segment mmap'd for fast appends; sealed segments mmap'd for fast reads. |
| **Pluggable segment store** | Active segment local, sealed segments in S3/GCS. Modern "lakehouse" engines. |

## When NOT to use

- **Logs that will always remain small** — a single file is simpler.
- **In-memory-only systems** — no disk concern.
- **Logs that must be readable as a single stream from byte 0** — segmentation makes you handle file boundaries (though most clients hide this).

---

## Real-world implementations

| System | Segment design |
|---|---|
| **Apache Kafka** | Per-topic-partition; size-based rotation (default 1 GB); time + size + compaction retention; sparse `.index` and `.timeindex` files; KIP-405 tiered storage. |
| **PostgreSQL WAL** | 16 MB segments in `pg_wal/`; rotated and archived. |
| **MySQL binlog** | Sequential numbered files; `max_binlog_size` rotation; `expire_logs_seconds` retention. |
| **Cassandra commitlog** | Configurable size; flushed-into-SSTable retention. |
| **RocksDB / LevelDB** | SSTable files at each LSM level; compaction merges. |
| **etcd WAL** | Segmented WAL files. |
| **HDFS edit logs** | Finalized edit log segments. |
| **Apache Pulsar / BookKeeper** | Ledgers = segments; tiered offload to S3. |
| **AWS Kinesis** | Internal segmentation hidden behind shard interface; trim horizon = LWM. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **LinkedIn, Confluent, Uber, Netflix, Pinterest, Shopify** | Run Kafka deployments at scale; rely on segmented log semantics. | ✅ Verified — engineering blogs |
| **PostgreSQL community** | WAL segmentation underpins streaming replication, archive_command, PITR backup tooling. | ✅ Verified — [Postgres docs on WAL](https://www.postgresql.org/docs/current/wal-intro.html) |
| **Google** | LevelDB / SSTable (open-source) generalized into BigTable's tablet structure and into RocksDB. | ✅ Verified — LevelDB / Bigtable papers |
| **Facebook / Meta** | RocksDB development; LSM-tree compaction at scale. | ✅ Verified — [RocksDB project, Facebook open source](http://rocksdb.org/) |

---

## Further reading

- Jay Kreps, *The Log: What every software engineer should know* (2013) — discusses segmented-log structure as the natural shape of any production log.
- Joshi, *Patterns of Distributed Systems*, "Segmented Log" pattern.
- Kafka documentation, [Section on log structure](https://kafka.apache.org/documentation/#log) — the most detailed public design discussion of a segmented log.
- Kleppmann, *Designing Data-Intensive Applications*, Ch 3 — SSTables, LSM-trees, and the segmented-file approach to durable storage.
- *Database Internals*, Alex Petrov, Ch 7 — log-structured storage in depth.
- O'Neil et al., *The Log-Structured Merge-Tree (LSM-Tree)* (1996) — the seminal paper that turned segmented logs into a primary storage architecture.

---

*Diagram sources: [`../diagrams/src/segmented-log-structure.d2`](../diagrams/src/segmented-log-structure.d2), [`../diagrams/src/segmented-log-read.d2`](../diagrams/src/segmented-log-read.d2).*
