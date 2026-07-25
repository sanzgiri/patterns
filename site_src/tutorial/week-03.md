# Week 3 — Sharding, Replication, and Consistency

!!! abstract "Learning goal"
    Choose a partition key, recognize hot partitions, select a replication strategy, and state a precise consistency contract.

## Lesson: distribute by query and write shape

A good partition key distributes load, keeps common queries local, and has stable ownership. A bad one creates hot spots, cross-shard joins, or difficult rebalancing.

- **Hash/consistent hashing:** good point lookups and distribution.
- **Range partitioning:** good ordered scans, but hot ranges are possible.
- **Geohash:** useful for geographic locality, but density is uneven.

Replication is not the same as consistency. A follower may contain the data eventually while still returning a stale answer immediately after a successful write.

Useful contracts include:

- read-your-writes;
- monotonic reads;
- bounded staleness;
- linearizability;
- eventual convergence.

Choose the weakest contract that satisfies the product requirement.

## Worked example: global inventory and orders

```text
regional orders ──> order shard ──> inventory authority
                                      │
                                      └── replicated log / CDC
                                               ↓
                                      browsing projections
```

Browsing can use eventually consistent projections. Checkout must revalidate against the inventory authority. Do not let every region independently decrement the same stock counter unless the design explicitly accepts overselling or uses a bounded reservation scheme.

### Failure cases

- Hot product: partition reservation work or preallocate regional stock.
- Replica lag: browsing may be stale; checkout revalidates.
- Region partition: reject reservations, spend a bounded regional allocation, or accept reconciliation risk.
- Leader promotion: verify log state and fence the old leader.

## Practice: now answer

1. What makes a partition key good?
2. Why can a user see an older value immediately after writing?
3. Which parts of inventory can be eventually consistent?
4. What should happen to checkout during a region partition?
5. Why does quorum not automatically mean linearizability?

## Interview drill

Design a globally distributed key-value store. Clarify the consistency contract before naming the database. Then cover partitioning, replication, quorum, rebalancing, hot keys, repair, and network partitions.

!!! warning "Common traps"
    Do not claim that consistent hashing solves hot keys, that replication guarantees exactly-once behavior, or that “eventual consistency” is a sufficient product contract.
