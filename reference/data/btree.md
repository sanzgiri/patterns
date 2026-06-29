# B-Tree

**Aliases:** B+Tree (the variant used in nearly all real databases), Balanced Tree
**Category:** Data / Storage engine
**Sources:**
[Joshi — Patterns of Distributed Systems](https://martinfowler.com/articles/patterns-of-distributed-systems/) ·
Kleppmann *DDIA*, Ch 3 ·
[Comer, *The Ubiquitous B-Tree* (ACM Computing Surveys, 1979)](https://carlosproal.com/ir/papers/p121-comer.pdf)

---

## Problem

> [!TIP]
> **ELI5.** You have a sorted phone book with 10 million names. Looking up "Smith" by flipping through every page is awful. Even binary search on a single huge list works in memory but is awkward when the book is split across many physical pages on disk. You want an index that points you to the right page in just a few hops.

A database needs to store sorted keys and answer:

- **Point queries** — "find the value for key K" — fast.
- **Range scans** — "list all keys in [A, B]" — fast and in order.
- **Inserts, updates, deletes** — without rebuilding the world.

On a single machine with everything in memory, a balanced binary search tree solves this. But databases live on **disk**, where the unit of I/O is a page (typically 4–16 KB), and **random reads are 1000× slower than sequential reads**. A binary tree with thousands of nodes means thousands of random page reads — a query takes seconds instead of microseconds.

You need a tree structure that **matches the disk's granularity**: each node should be roughly one page, with hundreds of keys, so traversing from root to leaf takes a handful of page reads. The B-tree (and its near-universal variant, the B+tree) is that structure.

## How it works

> [!TIP]
> **ELI5.** A B-tree is a many-way sorted tree where each node is a disk page holding hundreds of keys plus pointers to child pages. To find a key, start at the root, find the right sub-range, descend to the child page, repeat. With hundreds of children per node, a 4-level tree holds **billions** of keys — every lookup is at most 4 page reads.

A B-tree is a **self-balancing search tree** in which every node holds many keys (and many children — much more than 2). The two key properties:

1. **All leaves are at the same depth** — the tree is perfectly balanced, so every lookup costs the same number of page reads (`O(log_b n)` where `b` is the branching factor).
2. **High branching factor** — each node holds dozens to hundreds of keys, so the tree is *shallow*. A 4-level B-tree with branching factor 200 holds up to 200⁴ = 1.6 billion keys.

![B-tree structure](../diagrams/svg/btree-structure.svg)

In the diagram, the **root node** contains separator keys [30, 70]. Looking up key 45: the root tells us 45 is between 30 and 70, so descend to the middle child. That internal node contains [40, 50, 60]; 45 is between 40 and 50, descend to the leaf containing keys [40–49]. We've made **3 page reads** to find a key in a tree of billions.

In the **B+tree variant** (used by every major SQL database — Postgres, MySQL/InnoDB, SQLite, Oracle, SQL Server), **all data lives in the leaves**; internal nodes contain only routing keys. Leaves are linked together in a sorted doubly-linked list — so range scans walk along leaf pages in sequence without re-descending the tree. This is enormously efficient: `SELECT * WHERE id BETWEEN 100 AND 200` is a single tree descent followed by a sequential walk.

### Writes: in-place update + WAL

The B-tree's distinguishing characteristic — and its key trade-off versus the LSM-tree — is that **updates happen in place**. Modifying a row means finding its leaf page, modifying the bytes there, and writing the page back to disk.

![B-tree write path](../diagrams/svg/btree-write.svg)

When the engine performs an insert, update, or delete:

1. **Find the leaf page** by descending from the root. Each level is typically a cache hit (root and upper internal nodes are usually hot); the leaf is the most likely miss.
2. **Write the change to the WAL first** ([wal.md](wal.md)). The WAL entry is `fsync`'d before the operation is acknowledged — that's where durability comes from.
3. **Modify the page in memory** (in the page cache or buffer pool). The page is now "dirty"; it differs from its disk version.
4. **Acknowledge the commit** to the client.
5. **Background checkpoint** flushes dirty pages to their on-disk locations later. This is the expensive random-I/O part, but it's batched across many transactions, amortizing the cost.

The "in-place" character means writes are O(log n) page reads + WAL append + later random page write. Reads are O(log n) page reads. Both are predictable and bounded.

### Splits and merges

If an insert makes a leaf page **overflow** (no space for the new key), the page is **split**: half the keys go to a new sibling page, and the parent gets a new separator key. If the parent overflows, it splits too, recursively — potentially all the way up to a new root. Splits are infrequent (one per N inserts) and the cost is amortized to nearly free per insert.

Deletes can similarly trigger **merges** when leaves become too empty. In practice, most database engines defer merges (they leave "free space" in pages and reuse it for future inserts) because aggressive merging causes write amplification with little benefit.

### Concurrent access

B-trees in production databases support **many readers and writers concurrently**, using either:
- **Lock coupling / crabbing**: hold a lock on the parent while acquiring the child, then release the parent. Lock chains move down the tree like a crab's legs.
- **Latch-free / optimistic**: techniques like Bw-tree (Microsoft) or modern InnoDB use atomic CAS operations on individual pages, allowing high concurrency without traditional locking.

When combined with [MVCC](mvcc.md), the database can offer the "readers never block writers" guarantee — even though the B-tree pages themselves are physically modified, MVCC keeps logically older versions visible to in-flight readers.

### Trade-offs versus LSM-trees

The B-tree is the dominant choice for **read-heavy, latency-sensitive OLTP**: bank transactions, e-commerce checkouts, ERP. Reads are point-lookup-fast and predictable. The cost is **write amplification**: every update writes the page (~8 KB) plus a WAL entry, even for a tiny change.

The [LSM-tree](lsm-tree.md), by contrast, optimizes for **write-heavy workloads** by appending to in-memory structures and only later merging them on disk — sacrificing some read latency for much higher write throughput. The B-tree vs LSM-tree choice is one of the deepest in database engine design, and modern engines (CockroachDB, TiKV, FoundationDB) increasingly use LSM-tree underneath an SQL/transactional layer. Most legacy SQL databases (Postgres, MySQL/InnoDB, Oracle, SQL Server) stick with the B+tree because OLTP read patterns favor it.

A specialized variant, the **Bw-tree** (Microsoft Research, 2013), uses log-structured updates layered atop a B-tree structure, attempting to combine the strengths of both. It's used in Microsoft's Hekaton in-memory engine and in Azure Cosmos DB.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **B+tree** | Data only in leaves; leaves linked for fast range scan. The version everyone actually uses. |
| **B*-tree** | Keeps nodes 2/3 full instead of 1/2 (delayed splits) — fewer splits, more space-efficient. |
| **Bw-tree** | Lock-free, log-structured B-tree variant. Microsoft Hekaton, Azure Cosmos DB. |
| **Copy-on-Write B-tree** | LMDB, BoltDB — new versions written to new pages; root atomically swapped; gives snapshot isolation for free. |
| **Fractal Tree (TokuDB)** | Buffers writes at each internal node to delay propagation — better write performance, retired commercially but influential. |
| **R-tree** | Spatial generalization for multidimensional data (geographic indexes, PostGIS). |
| **Skip List** | Probabilistic balanced structure used in memory (Redis sorted sets) and in LSM memtables; conceptually related. |
| **LSM-tree** | The major alternative — see [lsm-tree.md](lsm-tree.md). |

## When NOT to use

- **Write-heavy workloads with sequential keys** — appending to a B-tree causes hot-page contention (e.g., timestamp-keyed inserts hammer the rightmost leaf). LSM-trees handle this gracefully.
- **Pure analytical workloads** — column-oriented formats (Parquet, ORC, Apache Arrow) beat B-trees for scans.
- **In-memory-only data** — a hash table or skip list is simpler and faster than a B-tree if you're not paging to disk.
- **Append-only event stores** — a [segmented log](segmented-log.md) is simpler and more natural.

---

## Real-world implementations

| System | B-tree variant |
|---|---|
| **PostgreSQL** | B+tree (default index type); GIN/GiST/BRIN for other workloads. |
| **MySQL / InnoDB** | B+tree (clustered primary, secondary indexes). |
| **Oracle, SQL Server, DB2** | B+tree. |
| **SQLite** | B+tree. |
| **MongoDB / WiredTiger** | B-tree (with LSM-tree as alternative engine). |
| **LMDB / BoltDB / etcd's bbolt** | Copy-on-write B+tree. |
| **Microsoft Hekaton / Azure Cosmos DB** | Bw-tree. |
| **CockroachDB / TiKV** | Use RocksDB (LSM) underneath, with B-tree-style range-key structure on top. |
| **Filesystems**: NTFS, HFS+, APFS, ext4, XFS | B-tree-based directory and metadata structures. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Every Postgres/MySQL/Oracle/SQL-Server-using company** | The default index in every major SQL DB is a B+tree. | ✅ Universal — see product docs |
| **Microsoft Azure** | Bw-tree in Hekaton (SQL Server in-memory) and Cosmos DB. | ✅ Verified — [*The Bw-Tree*, Microsoft Research (2013)](https://www.microsoft.com/en-us/research/publication/the-bw-tree-a-b-tree-for-new-hardware/) |
| **Apple, Google (Android), most mobile apps** | SQLite as on-device storage uses B+tree. | ✅ Verified — SQLite docs |
| **Linux kernel** | ext4, btrfs, xfs use B-tree-family structures for metadata. | ✅ Verified — kernel source |
| **Bitcoin Core, Ethereum clients** | Use LevelDB (LSM) but with B-tree-like Merkle Patricia tries on top. | ✅ Verified — protocol specs |

---

## Further reading

- Douglas Comer, *The Ubiquitous B-Tree* (ACM Computing Surveys, 1979) — the foundational survey. [PDF](https://carlosproal.com/ir/papers/p121-comer.pdf).
- Kleppmann, *Designing Data-Intensive Applications*, Ch 3 — B-trees vs LSM-trees side-by-side; one of the most-cited chapters in the book.
- Justin Levandoski, David Lomet, Sudipta Sengupta, *The Bw-Tree: A B-tree for New Hardware* (ICDE 2013) — Microsoft's lock-free B-tree. [PDF](https://www.microsoft.com/en-us/research/publication/the-bw-tree-a-b-tree-for-new-hardware/).
- *Database Internals*, Alex Petrov — Ch 2 + 4 give the most accessible modern treatment of B-tree internals.
- *The Internals of PostgreSQL*, Hironobu Suzuki (free online) — illustrated walkthrough of Postgres B+tree implementation.
- *Modern B-Tree Techniques*, Goetz Graefe (Foundations and Trends in Databases, 2011) — comprehensive 200+-page survey of B-tree research.

---

*Diagram sources: [`../diagrams/src/btree-structure.d2`](../diagrams/src/btree-structure.d2), [`../diagrams/src/btree-write.d2`](../diagrams/src/btree-write.d2).*
