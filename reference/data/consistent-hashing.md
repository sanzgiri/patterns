# Consistent Hashing

**Aliases:** Hash Ring, Ring-based Partitioning, Karger's Consistent Hashing
**Category:** Data distribution / Scalability
**Sources:**
[Karger, Lehman, Leighton, Levine, Lewin, Panigrahy — *Consistent Hashing and Random Trees* (STOC 1997)](https://www.akamai.com/site/en/documents/research-paper/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web.pdf) ·
[DeCandia et al. — *Dynamo: Amazon's Highly Available Key-value Store* (SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) ·
[Cassandra documentation: token ranges](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html) ·
[ByteByteGo — Consistent Hashing](https://blog.bytebytego.com/p/ep18-consistent-hashing-and-how-it) ·
microservices.io and Neo Kim mentions

---

## Problem

> [!TIP]
> **ELI5.** You have N servers and want to spread keys across them. The naive answer is `server = hash(key) % N`. Now add or remove one server (a routine cluster operation): suddenly almost every key maps to a different server. For a cache, that's a stampede — the whole cache effectively cold. For a sharded database, that's terabytes of data movement. **Consistent hashing** is a way to assign keys to servers such that adding or removing a server moves only ~1/N of the keys, not all of them.

In any distributed system with stateful nodes, you face this question: *given a key, which node owns it?*

- Distributed cache: which Memcached node stores `user:42`?
- Sharded database: which shard holds the row for `customer_id = 42`?
- Distributed hash table (DHT): which peer in a P2P network stores this file chunk?
- Load balancer with session affinity: which backend should serve this user's requests?

The naive answer — modulo hashing, `server = hash(key) % N` — has a fatal flaw:

![Modulo hashing vs consistent hashing: the rebalance](../diagrams/svg/consistent-hashing-vs-modulo.svg)

When `N` changes (add a server, remove a server, replace a failed one), almost every key remaps. For a 100-server cluster going to 101 servers, ~99% of keys move. For a cache, this means a near-total cache miss event — every request now misses the cache, slams the DB, the DB collapses, the site goes down. For a sharded database, it means moving terabytes of data across the network during the resize.

This isn't a theoretical concern; it kills clusters in practice. It was the specific problem Akamai needed to solve in the late '90s for their web caches, and the solution — **consistent hashing**, published by Karger et al. in 1997 — is now foundational infrastructure.

The trick: instead of mapping keys to servers via `key % N` (which couples every key's mapping to the value of N), arrange both servers and keys on a circular hash space (the *ring*). A key is owned by the next server clockwise from its position on the ring. Adding a server changes ownership only for keys in the *one arc* the new server's position covers — about K/N keys instead of nearly all K keys.

## How it works

> [!TIP]
> **ELI5.** Imagine a circular clock face numbered 0 to a huge number (the *hash space*). Place each server at multiple positions on the clock based on `hash(server_name)`. To find a key's server: hash the key to get a position on the clock, then walk clockwise until you hit a server — that server owns the key. If you add a new server, it claims keys only in its small section of the clock; everything else stays where it was.

The basic ring:

![Consistent hashing ring with vnodes](../diagrams/svg/consistent-hashing-ring.svg)

The core algorithm:

1. **Hash space**: a large integer range (typically 0 to 2⁶⁴-1), conceptually arranged in a circle (the "ring").
2. **Server placement**: each server is hashed (usually with multiple distinct labels per server) and placed on the ring at the resulting positions.
3. **Key lookup**: hash the key to get a ring position; walk clockwise to find the first server position. That server owns the key.
4. **Adding a server**: hash and place it on the ring. It now owns the keys between its position and the previous (counter-clockwise) server's position. Only those keys need to move (from the next clockwise server, where they were previously owned).
5. **Removing a server**: its arc of the ring is taken over by the next clockwise server. Only keys in that arc need to be redistributed.

The math: adding 1 server to N moves on average **K/N** keys (out of K total). Removing 1 server from N moves on average K/N keys. That's a vast improvement over modulo's near-K.

### Virtual nodes (vnodes): essential in practice

With just one ring position per server, you get **unbalanced load**:

- Random hash positions don't divide the ring evenly. Some servers get small arcs, some get huge arcs.
- Variance between server loads can be 2-3x even with a dozen servers.
- Adding/removing a server only affects its single neighbor, concentrating the rebalance load on one node.

The fix: **virtual nodes (vnodes)**. Each physical server is hashed multiple times — typically 100 to 500 — with different labels (`server1-0`, `server1-1`, ..., `server1-499`), and each hash places it at a different ring position.

Effects:
- **Smoother distribution**: each physical server owns many small arcs spread around the ring; loads even out.
- **Weighted servers**: a server with twice the capacity gets twice as many vnodes.
- **Distributed rebalance**: adding a new physical server takes a few keys from many existing servers, not all from one neighbor.

Vnode count is a tuning parameter. Too few → uneven load. Too many → bigger routing tables, more vnode-management overhead. 100-500 is the typical range; Cassandra defaults to 256 per node.

### The Dynamo lineage

The 1997 Karger paper was theoretical; **Amazon's Dynamo paper (2007)** brought consistent hashing to mainstream system design. Dynamo introduced:

- The ring + vnode model that everyone copies.
- **Preference lists**: instead of just "the next server," replicate to the next N servers clockwise. This is how Cassandra, Riak, and DynamoDB handle replication on top of consistent hashing.
- **Sloppy quorum + hinted handoff**: when a target server is unavailable, write to the next available one and "hand off" the write when the target returns.

Cassandra, Riak, ScyllaDB, and parts of Amazon DynamoDB are all descendants of Dynamo's design.

### Where it's used

- **Distributed caches**: Memcached clients (with consistent hashing) — Twitter's twemproxy, Mcrouter, Couchbase. Adding/removing cache nodes doesn't wipe most of the cache.
- **NoSQL databases**: Cassandra, Riak, ScyllaDB use consistent hashing for partitioning. DynamoDB does too (under the hood).
- **CDNs**: Akamai uses it to route requests to edge servers (this is what the original 1997 paper was for).
- **Distributed hash tables (DHT)**: Chord, Pastry, Kademlia (used in BitTorrent's DHT) — variants of consistent hashing.
- **Service meshes / load balancers**: Envoy, NGINX, HAProxy support ring-hash load balancing for stateful upstream selection (sticky-by-key without affinity tables).
- **Sharded systems generally**: any time you need stable key-to-node mapping that survives membership changes.

### What it doesn't solve

Consistent hashing is great for **stable key-to-node mapping**, but it's not magic:

- **Doesn't prevent hotspots from skewed keys.** If one key gets 50% of traffic, it lives on one server regardless of how clever the hashing is. Mitigations: client-side replication, request batching, or designing keys so traffic spreads.
- **Doesn't handle range queries well.** Consistent hashing scatters sequential keys (`order_1`, `order_2`, ..., `order_1000`) randomly across servers. For "give me all orders from yesterday," range partitioning is better.
- **Doesn't solve replication.** It tells you which node owns a key; replication (Dynamo's preference list, Cassandra's RF=3) is layered on top.
- **Adding/removing nodes still costs data movement.** It's just bounded — K/N instead of K. For a 1 TB cluster of 100 nodes, that's still 10 GB to move on add.
- **Vnode count is a real tuning lever.** Too few → uneven. Too many → routing-table bloat.

### Compared to alternatives

A few alternatives exist for the "which node owns this key?" question:

![Partitioning strategies](../diagrams/svg/partitioning-strategies.svg)

- **Modulo hashing**: simple but useless when N changes.
- **Consistent hashing**: O(log N) lookup via sorted ring positions; needs vnodes for balance.
- **Rendezvous hashing (HRW)**: O(N) lookup; no vnode data structure needed; very even distribution. Used by GitHub Spokes, some CDNs.
- **Range partitioning**: supports range queries; hotspot risk on sequential keys.
- **Hash + range hybrid**: shard by hash of a high-cardinality prefix, then range-scan within the shard. Used by DynamoDB (partition key + sort key) and Cassandra (composite primary key).
- **Lookup table / directory**: a metadata service tells you which shard owns which key. Used by Vitess, MongoDB router, HBase region servers. More flexible but introduces a metadata-service dependency.

The right choice depends on access patterns: point lookups → hashing; range scans → range or hybrid; needs flexibility → directory.

### Rendezvous hashing: worth knowing

Rendezvous hashing (Highest Random Weight) achieves the same key-movement properties as consistent hashing with a different algorithm:

For each server, compute `hash(key + server_id)`. The server with the highest hash value wins. To add a server, you start computing its hashes too; keys for which the new server scores highest now belong to it.

- O(N) per lookup (vs O(log N) for ring) — fine for moderate N.
- No vnode data structure to maintain.
- Even distribution without vnodes.
- Used in GitHub's Spokes (git server load balancing), some video streaming, some gRPC client load balancers.

For small clusters and simple code, rendezvous hashing is often preferable to consistent hashing.

### Implementation tips

- **Use a well-mixed hash function** (MurmurHash3, xxHash, FNV) — not crypto hashes (overkill, slow) and not `hashCode()` (poor distribution).
- **Pre-build the sorted ring** — use a sorted map or skiplist for O(log N) lookup.
- **Pick vnode count by simulation** — generate a representative workload and measure load variance.
- **Handle "next clockwise from position 0"** — wrap around the ring.
- **For replication**: take the next N distinct *physical* servers, not just the next N vnodes (which might be the same physical server).
- **Persist the ring or compute it deterministically** — clients and servers must agree on the mapping.

### Real-world implementations

- **Cassandra**: token-based consistent hashing with vnodes.
- **DynamoDB**: consistent hashing for partition key; sort key for range within a partition.
- **Memcached clients** (ketama algorithm — libmemcached, mcrouter, twemproxy).
- **Riak**: Dynamo-style consistent hashing.
- **Envoy / NGINX**: `ring_hash` load balancing policy.
- **Akamai**: the original use case; CDN edge routing.
- **Kademlia DHT** (BitTorrent, IPFS): variant for P2P networks.
- **Ceph** (CRUSH algorithm): consistent-hashing-like placement for object storage.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Ring + vnodes (Dynamo-style)** | The dominant variant; smooth distribution. |
| **Ketama** | Specific Memcached algorithm; consistent hashing with MD5. |
| **Rendezvous (HRW) hashing** | Alternative algorithm; no ring; O(N) lookup. |
| **Jump consistent hash** | Lai Wei & Lamping (2014); very fast, but only supports key-to-bucket. |
| **Maglev hashing** | Google's consistent hashing variant; uniform load, fast lookup. |
| **CRUSH (Ceph)** | Tree-aware placement; consistent-hashing-like. |
| **Range partitioning** | Alternative when range queries matter. |
| **Directory-based** | Lookup-table alternative; more flexible. |

## When NOT to use

- **Range query workloads** — use range partitioning instead.
- **Tiny cluster (≤3 nodes)** — the savings are not worth the complexity.
- **When keys are highly skewed** — hashing alone won't save you from a 50%-traffic key.
- **When you need cross-key transactions** — keys living on different nodes complicate transactions.

---

## Real-world implementations

| Tool | Notes |
|---|---|
| **Cassandra** | Token-based consistent hashing with vnodes (default 256). |
| **Riak** | Dynamo-style ring. |
| **DynamoDB** | Internal consistent hashing for partition key. |
| **ScyllaDB** | Cassandra-compatible; consistent hashing. |
| **Memcached (clients)** | Ketama, twemproxy, mcrouter, libmemcached. |
| **Couchbase** | Vbucket-based (similar concept). |
| **Envoy** | `ring_hash` load balancer. |
| **NGINX Plus / HAProxy** | Ring-hash upstream selection. |
| **Akamai** | Original use case; CDN routing. |
| **BitTorrent DHT (Kademlia)** | XOR-based DHT; variant. |
| **IPFS / libp2p** | Kademlia-based DHT. |
| **Apache Ignite, Hazelcast** | Distributed data grids use consistent hashing for partition assignment. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Amazon** | Dynamo and DynamoDB; canonical example since 2007 paper. | ✅ Verified — [Dynamo SOSP 2007 paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) |
| **Akamai** | Original use case in the 1997 Karger paper. | ✅ Verified — [Karger et al. STOC 1997](https://www.akamai.com/site/en/documents/research-paper/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web.pdf) |
| **Facebook** | Mcrouter for Memcached pools uses consistent hashing. | ✅ Verified — [Mcrouter open source](https://github.com/facebook/mcrouter) |
| **Twitter / X** | Twemproxy with consistent hashing for Memcached pools. | ✅ Verified — Twitter Engineering blog |
| **Netflix** | EVCache uses consistent hashing. | ✅ Verified — Netflix Tech Blog, EVCache GitHub |
| **Apple** | Cassandra at massive scale; consistent hashing for partitioning. | ✅ Verified — Apple's Cassandra Summit talks |
| **Discord** | Cassandra-based message store; consistent hashing. | ✅ Verified — Discord Engineering blog |
| **GitHub** | Spokes uses Rendezvous hashing (a relative). | ✅ Verified — GitHub Engineering blog |
| **Google** | Maglev load balancer uses consistent-hashing variant. | ✅ Verified — Maglev NSDI paper |

---

## Further reading

- Karger et al., *Consistent Hashing and Random Trees* (STOC 1997) — the original.
- DeCandia et al., *Dynamo: Amazon's Highly Available Key-value Store* (SOSP 2007) — the influential application.
- Lamping & Veach, *A Fast, Minimal Memory, Consistent Hash Algorithm* (Jump hash) — alternative.
- Eisenbud et al., *Maglev: A Fast and Reliable Software Network Load Balancer* (NSDI 2016) — Google variant.
- Cassandra documentation — concrete implementation.
- *Designing Data-Intensive Applications* (Kleppmann), Ch 6 on partitioning.
- Hugo Sereno Ferreira's blog on consistent hashing — clear visual explanations.

---

*Diagram sources: [`../diagrams/src/consistent-hashing-ring.d2`](../diagrams/src/consistent-hashing-ring.d2), [`../diagrams/src/consistent-hashing-vs-modulo.d2`](../diagrams/src/consistent-hashing-vs-modulo.d2), [`../diagrams/src/partitioning-strategies.d2`](../diagrams/src/partitioning-strategies.d2).*
