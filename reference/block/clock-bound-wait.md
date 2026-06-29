# Clock-Bound Wait (Commit Wait)

**Aliases:** Commit Wait, TrueTime Wait, Uncertainty Wait
**Category:** Building block (timestamps)
**Sources:**
[Joshi — Patterns of Distributed Systems](https://martinfowler.com/articles/patterns-of-distributed-systems/) ·
Kleppmann *DDIA*, Ch 8 (clocks) + Ch 9 ·
[Corbett et al., *Spanner: Google's Globally-Distributed Database* (OSDI 2012)](https://research.google/pubs/pub39966/)

---

## Problem

> [!TIP]
> **ELI5.** You commit a transaction at "time T." Your clock might be slightly off — maybe true time is actually T+5ms or T-3ms. Another data center might commit at "time T-2ms" — but from the *real-world* perspective, they happened in a different order. If your clocks are uncertain, *you can't trust timestamps to give external consistency* without waiting out the uncertainty.

A distributed database that uses timestamps to order transactions (a [Hybrid Logical Clock](hybrid-logical-clock.md) or any wall-clock-based scheme) has a deep problem: **clocks lie**. NTP synchronization typically keeps clocks within a few milliseconds, but the *uncertainty* is real — you don't know the true time, only an interval that contains it. Without machinery to handle this uncertainty, the database can violate **external consistency**:

> If transaction T1 commits at timestamp t1, and a client (knowing T1 committed) starts T2 that observes T1's effects, then T2's commit timestamp must be greater than t1.

This is the property humans expect. "If I see that I paid the bill, the receipt should be timestamped after the payment." Naive systems break it: T1 commits at t1=1000ms on node A whose clock might really be 1005ms. Client gets the ack and starts T2 on node B whose clock is at 998ms (actually 999ms in real time). B assigns t2=998ms. From clock timestamps, T2 looks like it happened *before* T1 — even though it actually happened after, in real time, and the client knows it.

You need a way to guarantee that **the assigned commit timestamp is in the past by the time any subsequent observer can see the commit**. The clock-bound wait is the mechanism.

## How it works

> [!TIP]
> **ELI5.** When committing, pick a timestamp slightly in the future (the **latest** possible "now" given clock uncertainty). Replicate the commit to other nodes. Then **deliberately wait** until the real clock has definitely passed that timestamp before telling the client the commit is done. By the time the client sees the ack, no clock anywhere can produce a smaller timestamp.

The clock-bound wait, also called **commit wait**, was introduced by Google's Spanner (2012) as part of its **TrueTime** mechanism. The protocol assumes a special API:

- `TT.now()` returns an interval `[earliest, latest]` — both lower and upper bounds on the true current time. The width `ε = latest − earliest` is the **clock uncertainty**, typically 1–7ms in Spanner.

A commit then proceeds:

![Clock-bound wait protocol](../diagrams/svg/clock-bound-wait.svg)

1. **Pick the commit timestamp** as `commit_ts = TT.now().latest` — the latest possible true time *right now*. This ensures the timestamp is at or after the actual current time.
2. **Replicate the commit** via Paxos (or whatever consensus protocol). This takes some real time — typically a few milliseconds for intra-DC replication.
3. **Wait** until `TT.now().earliest > commit_ts` — that is, wait until the *earliest* possible true time has passed `commit_ts`. Equivalently: wait until "no possible clock anywhere in the system can still be earlier than `commit_ts`."
4. **Only then ack the client**. By the time the client knows the commit succeeded, real time has definitely advanced past `commit_ts`.

The wait length is approximately `ε` — the uncertainty bound. In Spanner's case, with GPS + atomic clocks, ε is typically 1–7ms, so the wait is short. But it is **real wasted time** — during commit-wait, the leader is doing nothing useful, just waiting for the clock to catch up. This cost is the fundamental price of getting external consistency without expensive cross-DC coordination per read.

### Why the wait gives external consistency

The mechanics work because of an invariant:

![TrueTime uncertainty and commit wait](../diagrams/svg/truetime-uncertainty.svg)

After commit-wait completes:
- `commit_ts` has been assigned and replicated.
- Real time is *definitely* `>= commit_ts` (because we waited until `TT.now().earliest > commit_ts`).
- Therefore any *future* timestamp assigned by any node will satisfy `new_ts > commit_ts` — because every node's clock is bounded within its own `[earliest, latest]` interval, and the earliest possible time anywhere is now strictly greater than `commit_ts`.

The client gets the ack and starts a follow-up transaction. That transaction's commit timestamp is necessarily greater than `commit_ts`. External consistency: preserved.

### What TrueTime actually requires

Spanner's TrueTime infrastructure is impressive engineering:

- **GPS receivers** in every datacenter — they provide accurate time from satellites.
- **Atomic clocks** as backup — for the rare hours when GPS is unavailable.
- **Multiple time-masters** per region — providing redundancy and accuracy via NTP-style protocols.
- **Time-slave daemon** on every machine — runs a continuous protocol with the masters and computes `TT.now()`'s `[earliest, latest]` interval based on the round-trip latency and last sync time.

The result is `ε` of ~1–7ms in normal operation, growing to tens of ms in pathological cases. Spanner can use this directly for commit wait.

### Without TrueTime: HLC + bounded uncertainty

For databases that don't have GPS-and-atomic-clock infrastructure, **clock-bound wait can still be applied with HLCs** — just less aggressively, and with explicit assumptions about NTP accuracy. CockroachDB does this: it assumes `MAX_OFFSET` of 500ms between any two nodes (configurable; smaller is more efficient but riskier), and on commit, it waits long enough that any reasonably-skewed clock has passed the commit timestamp.

The trade-off: CockroachDB's commit wait is much longer than Spanner's (the wait equals MAX_OFFSET, ~500ms vs Spanner's ~5ms), making cross-DC transactions slower. To mitigate, CockroachDB doesn't use commit wait for all transactions — only for those that need strict external consistency. Most workloads get "snapshot isolation" or "serializable" with a weaker (but still safe) timestamp scheme.

### The economic argument for TrueTime hardware

Why would anyone invest in GPS + atomic clocks? Because of what it lets you do *without coordination*:

- **Stale reads at any timestamp**: any replica can serve a read at timestamp T, returning the consistent snapshot at T — *without* asking the leader or any other node. With commit wait, T-snapshots are coherent across the cluster.
- **Snapshot isolation across the planet** without 2PC per read.
- **Cheap follower reads** in geo-distributed deployments — read from your local datacenter without coordinating across the world.

For a system serving billions of reads per second across continents, eliminating coordination per read is worth the hardware cost — perhaps the most expensive infrastructure investment in distributed database history, and possibly the most strategically valuable.

### Alternatives

If you can't afford TrueTime and don't want HLC's MAX_OFFSET wait:

- **Single TSO (Timestamp Oracle)**: have one process (replicated for HA) hand out timestamps. Used by TiDB. Sidesteps the clock problem but introduces a single coordinator (TSO bottleneck).
- **Optimistic transactions with retries**: assume no commit-wait, detect violations at validation time, retry. FoundationDB's approach.
- **Causal-only consistency**: don't aim for external consistency; provide causal consistency (HLC alone gives this). Many "eventually consistent with read-your-writes" systems.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **TrueTime commit-wait (Spanner)** | The original — hardware-bounded clocks, ~5ms wait. |
| **MAX_OFFSET wait (CockroachDB)** | NTP-based with assumed bound; longer wait, no hardware. |
| **Per-operation conditional wait** | Wait only for transactions that need external consistency; others use weaker timestamps. |
| **TSO (Timestamp Oracle)** | Replace clock-bound wait with a single source of timestamps. TiDB. |
| **Optimistic detection + retry** | Don't wait; detect and retry on conflict. FoundationDB. |
| **Vector clock + causal-only** | Skip external consistency; use vector clocks for what causal info you need. Dynamo-style. |

## When NOT to use

- **No external-consistency requirement** — causal consistency (HLC + no wait) suffices for many apps.
- **Single-leader systems** — the leader's timestamp is monotonic; no cross-node disagreement to resolve.
- **Workloads that tolerate brief inconsistency** — saves the commit-wait latency.
- **Without bounded-uncertainty clocks** — using commit-wait without knowing your `ε` means waiting for an arbitrary length, which can dwarf the actual transaction time.

---

## Real-world implementations

| System | Mechanism |
|---|---|
| **Google Spanner** | TrueTime + commit wait (1–7ms). Originator. |
| **Google Cloud Spanner** | Same as internal Spanner. |
| **CockroachDB** | HLC + MAX_OFFSET-based commit wait (default 500ms; configurable). |
| **YugabyteDB** | HLC; commit wait optional per workload. |
| **TiDB / TiKV** | TSO (single timestamp oracle) — different approach, same goal. |
| **FoundationDB** | Optimistic conflict detection — sidesteps commit wait entirely. |
| **Apache Kudu** | HLC-style for snapshots; no commit wait. |
| **YDB (Yandex)** | Spanner-like with NTP-based wait. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Google** | Spanner with TrueTime — backs Ads, Search, AdWords, internal systems. | ✅ Verified — [Spanner OSDI 2012](https://research.google/pubs/pub39966/); *Spanner: Becoming a SQL System* SIGMOD 2017 |
| **Google Cloud customers** | Anyone using Cloud Spanner relies on TrueTime + commit wait. | ✅ Verified — [Cloud Spanner docs](https://cloud.google.com/spanner/docs/true-time-external-consistency) |
| **Cockroach Labs and customers** | MAX_OFFSET-based variant in production. | ✅ Verified — [CockroachDB clock docs](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer) |
| **YugabyteDB users** | HLC + optional commit wait. | ✅ Verified — YugabyteDB docs |

---

## Further reading

- Corbett et al., *Spanner: Google's Globally-Distributed Database* (OSDI 2012) — the foundational paper. [PDF](https://research.google/pubs/pub39966/).
- *Spanner: Becoming a SQL System* (SIGMOD 2017) — the production update.
- Kleppmann, *Designing Data-Intensive Applications*, Ch 8 (clocks, snapshot isolation across nodes).
- Joshi, *Patterns of Distributed Systems*, "Clock-Bound Wait" pattern.
- CockroachDB blog, *Living without atomic clocks* — explains how to apply commit-wait without TrueTime infrastructure.
- *Google's Globally-Distributed Database, Five Years Later* — retrospective talks from Spanner team.
- Daniel Abadi, *Spanner vs Calvin: Distributed Consistency at Scale* — comparative analysis of consistency mechanisms.

---

*Diagram sources: [`../diagrams/src/clock-bound-wait.d2`](../diagrams/src/clock-bound-wait.d2), [`../diagrams/src/truetime-uncertainty.d2`](../diagrams/src/truetime-uncertainty.d2).*
