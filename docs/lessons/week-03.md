# Week 3 — Sharding, replication, and consistency

## Outcome

By the end of this week, you should be able to select a partition key, recognize hot partitions, compare replication strategies, choose a consistency contract, and reason about failures without using “eventually consistent” as a complete answer.

## Patterns and material

- [Sharding](../../reference/data/sharding.md)
- [Consistent Hashing](../../reference/data/consistent-hashing.md)
- [Range Partitioning and Geohash](../../reference/data/range-partitioning-geohash.md)
- [Leader-Follower Replication](../../reference/data/leader-follower-replication.md)
- [Quorum](../../reference/block/quorum.md)
- [Replicated Log](../../reference/data/replicated-log.md)

## Session 1 — Partition-key reasoning

A good partition key distributes load, keeps common queries local, and has stable ownership. A bad key creates hot spots, cross-shard joins, or painful rebalancing.

Compare:

- hash/consistent hashing: good point lookups and distribution;
- range partitioning: ordered scans and time/range queries, but hot ranges are possible;
- geohash: locality-aware geographic queries, with uneven density at cities or events.

Never choose a key from the entity name alone. Start with the dominant query and write path.

## Session 2 — Replication and read semantics

Leader-follower replication gives one write authority and simpler ordering, but replicas lag and fail over. A client that writes to a leader and immediately reads a stale follower can observe time travel from its own perspective.

Possible contracts:

- **read-your-writes:** a user sees their own successful update;
- **monotonic reads:** a user does not move backward in observed versions;
- **bounded staleness:** replicas lag by a defined time or version bound;
- **linearizability:** each operation appears to happen atomically in real-time order;
- **eventual convergence:** replicas become equal if updates stop.

State the weakest contract that satisfies the product. Stronger consistency costs latency, availability, or operational complexity.

## Session 3 — Case: global inventory and orders

### Prompt

“Design inventory and order storage for a marketplace operating in multiple regions.”

Assume product browsing is global and read-heavy, orders are regionally created, inventory cannot be oversold beyond a small controlled reserve, and a region may become unreachable.

### Baseline design

```text
region A orders → regional order shard → inventory authority
region B orders → regional order shard → inventory authority
                         │
                  replicated log / CDC
                         ↓
                global browsing projections
```

Partition orders by region or merchant, but assign each inventory item an authority that serializes reservations. Replicate the resulting state to browsing projections. Do not let every region independently decrement the same stock counter unless you deliberately accept overselling or use a reservation/escrow scheme.

### Failure analysis

- **Hot product:** a single key becomes a hotspot; use partitioned reservation buckets, preallocated regional stock, or a queue that serializes updates.
- **Replica lag:** browsing may show stale availability; checkout must revalidate against the authority.
- **Region partition:** choose between rejecting reservations, spending a bounded regional allocation, or accepting reconciliation/oversell risk.
- **Replica promotion:** ensure the new leader has the required log prefix and fence the old leader.

## Session 4 — Interview defense

### Prompt

“Design a globally distributed key-value store.”

Answer in this order:

1. Define key/value size, read/write ratio, latency, durability, and consistency.
2. Choose hash or range partitioning from query shape.
3. Explain replication topology and failover.
4. Define quorum parameters and what overlap buys you.
5. Explain rebalancing and how clients find the right shard.
6. Walk through a write, a replica failure, and a network partition.
7. State observability: replica lag, quorum failures, hot partitions, rebalance movement, and stale-read rate.

### Common traps

- saying “consistent hashing solves scaling” without discussing hot keys;
- treating quorum as automatically linearizable;
- claiming exactly-once behavior from replication alone;
- ignoring rebalancing, repair, and tombstone/version cleanup;
- choosing multi-region active-active writes without conflict semantics.

## Deliverable and rubric

Submit a partition-key decision, consistency contract, replication diagram, and four failure walkthroughs. Score 0–3 on: partition reasoning, hot-key handling, consistency precision, replication/failover, repair/rebalancing, and trade-off communication.

