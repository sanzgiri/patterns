# CRDT (Conflict-Free Replicated Data Type)

**Aliases:** CmRDT, CvRDT, Eventually-Consistent Data Type, Mergeable Data Type
**Category:** Data / Coordination
**Sources:**
[Shapiro, Preguiça, Baquero, Zawirski — *A comprehensive study of Convergent and Commutative Replicated Data Types* (INRIA TR 2011)](https://hal.inria.fr/inria-00555588/document) ·
[Shapiro et al. — *Conflict-free Replicated Data Types* (SSS 2011)](https://link.springer.com/chapter/10.1007/978-3-642-24550-3_29) ·
[DeCandia et al. — *Dynamo* (SOSP 2007) — practical antecedent (vector clocks + siblings)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) ·
[Riak documentation on CRDTs](https://docs.riak.com/riak/kv/2.0.4/developing/data-types/index.html) ·
[Yjs / Automerge documentation](https://docs.yjs.dev/) ·
*Designing Data-Intensive Applications* (Kleppmann), Ch 5

---

## Problem

> [!TIP]
> **ELI5.** Two people edit the same Google-Doc-style document at the same time, possibly while offline. When they reconnect, both edits should merge automatically without losing anyone's work. Two shopping carts (your phone, your laptop) modified independently must merge cleanly when synced. Two replicas of a database accept writes simultaneously and must converge to the same state. The hard part: **no coordination, no leader, no locks** — you can't pause edits to ask "who's winning?". **CRDTs** are data types designed so that *every possible interleaving of edits converges to the same final state automatically*, by making the merge operation mathematically nice (commutative, associative, idempotent).

The motivation: many real systems need **multi-replica, multi-writer, no-coordination** semantics:

- **Collaborative editing** (Google Docs, Figma, Notion, Apple Notes): multiple users edit the same document concurrently, possibly with poor or no network.
- **Mobile/offline-first apps**: a phone makes edits while offline; eventually syncs with a server and other devices. Conflicts must merge.
- **Multi-master databases** (Riak, Cassandra in some modes, CosmosDB multi-region writes): writes can happen at any replica.
- **Eventually-consistent distributed systems** (Dynamo-style): no single leader; replicas independently process writes and must converge.
- **Edge / CDN scenarios**: state mutated at multiple edge POPs; eventually reconciled.

The naive approaches fail:

- **Lock + leader**: requires coordination on every write. High latency. Impossible offline. Single point of failure.
- **Last-writer-wins (LWW)**: pick a winner by timestamp. Simple but loses concurrent edits silently. (You both type a sentence; one disappears.)
- **Multi-value / "siblings"** (Dynamo): return all conflicting values to the application and let it resolve. Pushes hard work to every app developer.
- **Operational Transform (OT)**: used by Google Docs; works but requires a central coordinator to order operations. Hard to implement correctly.

**CRDTs** offer a fourth option: design the data type so that *the merge operation itself* guarantees convergence. If your merge is commutative (`A ⊔ B = B ⊔ A`), associative (`(A ⊔ B) ⊔ C = A ⊔ (B ⊔ C)`), and idempotent (`A ⊔ A = A`), then **all replicas that have seen the same set of updates — in any order, any number of times — converge to the same state**.

This is a mathematical guarantee. No coordination needed. The cost: you must design (or use) data types that conform to these properties. Not every data type can — but enough common types (counters, sets, maps, ordered lists, key-value stores, JSON documents) have CRDT formulations that the pattern is broadly applicable.

The formal theory was published by Marc Shapiro and Nuno Preguiça at INRIA in 2011, though earlier systems (Dynamo, Bayou, treedoc, WOOT) had been informally using the ideas for years.

## How it works

> [!TIP]
> **ELI5.** Design every data type so that merging two copies always gives the same answer regardless of order. Then any replica can apply edits locally, broadcast them lazily, and eventually all replicas agree — without ever coordinating in advance.

The core idea:

![CRDT concept](../diagrams/svg/crdt-concept.svg)

There are **two flavors** of CRDT that differ in what gets exchanged between replicas:

### State-based (CvRDT — Convergent)

Replicas periodically exchange their **full state**. A merge function combines two states into a "joined" state.

For convergence, the merge function must form a **join semi-lattice**:
- **Commutative**: `merge(A, B) = merge(B, A)`.
- **Associative**: `merge(merge(A, B), C) = merge(A, merge(B, C))`.
- **Idempotent**: `merge(A, A) = A`.

Each replica's state moves monotonically "up" the lattice; replicas eventually meet at a common state.

Trade-offs:
- Easy to reason about — just need to verify the merge is a lattice join.
- Robust to unreliable networks — duplicate or out-of-order delivery is fine.
- State can grow large; transferring full state is bandwidth-heavy. Mitigated with deltas (delta-CRDTs).

### Operation-based (CmRDT — Commutative)

Replicas exchange **operations** (e.g., "add x", "remove y") via reliable broadcast with causal delivery (vector clocks). Operations must commute.

Trade-offs:
- Smaller messages (just the op, not the state).
- Requires reliable causal delivery (more infrastructure).
- Operations themselves must be designed carefully to commute.

In practice, modern CRDT libraries (Yjs, Automerge) are hybrid — they send delta operations but use state-based reasoning for correctness.

### Common CRDT types

The CRDT zoo includes types for most common needs:

![CRDT catalog](../diagrams/svg/crdt-catalog.svg)

**G-Counter (grow-only counter)**: each replica increments only its own slot of a per-replica counter. Value = sum of all slots. Merge = elementwise max. Trivially commutative, associative, idempotent.

```
Replica A: {A: 5, B: 0, C: 0}
Replica B: {A: 0, B: 3, C: 0}
Replica C: {A: 0, B: 0, C: 7}
After merge: {A: 5, B: 3, C: 7} → value = 15
```

**PN-Counter (positive-negative)**: a pair of G-counters (one for increments, one for decrements). Value = P-counter - N-counter.

**G-Set (grow-only set)**: add-only set; merge = union. Items can never be removed.

**2P-Set (two-phase set)**: two G-sets (added, removed); element "in set" if added but not removed. Can't re-add after remove.

**OR-Set (observed-remove set)**: tags each add with a unique ID; remove only removes adds you've observed. Solves the "concurrent add-after-remove" problem of 2P-Set. The most practical mutable set CRDT.

**LWW-Register**: each write has a timestamp; on merge, keep highest-timestamp value. Simple but loses concurrent writes (the lower-timestamp one is discarded). Use only when concurrent writes are rare.

**MV-Register (Multi-Value)**: on concurrent writes, keep all values as "siblings"; app resolves. (Riak's default conflict-resolution model.)

**RGA, Y.js, Automerge — ordered sequence CRDTs**: support concurrent inserts and deletes at any position in an ordered list. The hard case — used by collaborative text editors. Each character gets a unique ID; operations preserve causal order.

**OR-Map / JSON CRDTs**: maps where keys can be added/removed, values are themselves CRDTs. Automerge and Yjs compose all the above into arbitrary JSON documents — supporting collaborative editing of nested arbitrary structures.

### A worked example: collaborative counter

Three replicas of a "likes counter":

```
Initial:    A: {A:0, B:0, C:0}    B: {A:0, B:0, C:0}    C: {A:0, B:0, C:0}

Alice clicks like on A:  A becomes {A:1, B:0, C:0}
Bob clicks like on B:    B becomes {A:0, B:1, C:0}
Carol clicks twice on C: C becomes {A:0, B:0, C:2}

(Network is partitioned for a while. Each replica continues locally.)

Alice → A: {A:3, B:0, C:0}      (two more likes)
Bob → B:   {A:0, B:1, C:0}      (no change)
Carol → C: {A:0, B:0, C:5}      (three more likes)

(Network heals. Replicas exchange states.)

A sees B: merge({A:3, B:0, C:0}, {A:0, B:1, C:0}) = {A:3, B:1, C:0}
A sees C: merge({A:3, B:1, C:0}, {A:0, B:0, C:5}) = {A:3, B:1, C:5}

Likewise B and C arrive at:                       {A:3, B:1, C:5} → value = 9
```

All replicas converge. No coordination. No conflicts. No lost updates. The value is the unique correct total — 9 likes.

### Where CRDTs are used in production

- **Riak** (open-source NoSQL): CRDTs as native data types (counters, sets, maps, registers).
- **Redis Enterprise** (Active-Active): CRDT-based geo-distributed multi-master.
- **Azure CosmosDB**: multi-region write conflict resolution uses CRDTs.
- **Yjs** and **Automerge**: libraries powering collaborative editing in many apps (Figma's offline mode, Notion, Linear, Mattermost, Tldraw, Excalidraw).
- **Apple Notes**: collaborative editing uses internal CRDTs (CRDB).
- **Facebook Apollo**: CRDT-based cache.
- **TomTom, Roblox, Soundcloud** (per public talks): CRDT replication for various features.
- **Slack messaging**: uses CRDT-based reconciliation for message ordering during reconnect.

The collaborative-editing space is dominated by two libraries:
- **Yjs**: high-performance, smaller deltas, more deployed.
- **Automerge**: more JSON-friendly, easier API for app developers.

Both are used in production by many real-time collaborative apps.

### When CRDTs are the right tool

CRDTs shine when:

- **Multi-master writes** without coordination are essential.
- **Offline operation** with eventual sync is a requirement.
- **Multiple geo-regions** with low-latency local writes.
- **Collaborative editing** is the use case.
- **A finite set of operations** (the CRDT types) covers your needs.

They're the wrong tool when:

- **You need transactional consistency** across data items (CRDTs are eventually consistent per-item, not globally consistent).
- **Your data type doesn't have a clean CRDT formulation** (complex business invariants, arbitrary computations).
- **Coordination is acceptable**: a leader-based system is much simpler.
- **Storage matters a lot**: many CRDTs need tombstones or metadata that grow over time.
- **Conflict semantics matter to users**: "merging" two simultaneously-edited sentences may produce garbled text that auto-resolution can't fix; OT (used by Google Docs) is better in some user-facing scenarios.

### Trade-offs

The advantages:
- **No coordination** — works offline, across geo-regions, with weak consistency.
- **Automatic merge** — no manual conflict resolution needed.
- **Mathematical guarantees** — convergence is proven, not best-effort.
- **Compositional** — CRDTs combine into larger CRDTs.
- **Mature libraries** — Yjs, Automerge, RiakKV, OrbitDB are production-ready.

The disadvantages:
- **Metadata overhead** — tombstones, vector clocks, unique IDs per element accumulate.
- **Garbage collection of tombstones** is hard (you can never be sure all replicas have seen a delete).
- **Application invariants beyond per-key correctness are not preserved.** "Account balance ≥ 0"? CRDTs can't enforce that across concurrent withdrawals.
- **Some operations don't have natural CRDT versions**: "rename this directory" in a filesystem CRDT is famously hard.
- **Sequence CRDTs (text)** have subtle behaviors around concurrent splits/joins.

### CRDTs vs OT

A common comparison in collaborative editing:

| | OT (Operational Transform) | CRDT |
|---|---|---|
| Coordination | Central server orders operations | Fully decentralized |
| Algorithm | Transforms ops based on what others did | Designs ops to commute |
| Implementation | Famously tricky (Google Docs took years) | Library-friendly |
| Network requirement | Reliable in-order delivery to server | Eventual delivery |
| Used by | Google Docs, Etherpad | Figma offline, Yjs apps, Automerge apps |

OT has the edge for richer user-intent preservation in text; CRDTs have the edge for offline, decentralization, and ease of implementation. Modern systems often pick CRDT for new development.

### CRDTs vs Saga, 2PC, Paxos

CRDTs solve a fundamentally different problem than [2PC](../coord/2pc.md), [Paxos](../coord/paxos.md), or [Saga](saga.md):

- 2PC/Paxos: serialize operations across replicas, sacrifice availability during partitions.
- Saga: handle long-running multi-service transactions with compensations.
- CRDT: accept independent local operations, converge by construction, never block.

They occupy different points on the CAP spectrum. CRDTs are the AP corner with eventual consistency.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **State-based (CvRDT)** | Exchange full state; merge function. |
| **Operation-based (CmRDT)** | Exchange ops; need causal delivery. |
| **Delta-state CRDT** | Exchange deltas instead of full state. |
| **G-Counter, PN-Counter** | Counters. |
| **G-Set, 2P-Set, OR-Set** | Sets. |
| **LWW-Register, MV-Register** | Single-value registers. |
| **OR-Map, JSON CRDT** | Composable maps. |
| **RGA, Logoot, WOOT, Yjs, Automerge** | Ordered sequence CRDTs. |
| **Vector clock + siblings (Dynamo)** | Pre-CRDT pattern; conflict-detection but manual merge. |
| **OT (Operational Transform)** | Alternative for collaborative editing; needs central coordinator. |

## When NOT to use

- **Transactional invariants required** ("account balance ≥ 0").
- **Coordination is acceptable** — leader-based is simpler.
- **Data type has no clean CRDT formulation**.
- **Storage is constrained** — tombstones / metadata grow.
- **User-visible merge semantics matter** — text CRDTs can produce surprising results in conflicts.

---

## Real-world implementations

| Tool | Notes |
|---|---|
| **Yjs** | High-performance JavaScript/WASM CRDT library; widely used. |
| **Automerge** | JSON-CRDT library; Rust core, multiple bindings. |
| **Riak** | Native CRDT data types in NoSQL DB. |
| **Redis Enterprise (Active-Active)** | CRDT-based geo-distributed. |
| **Azure CosmosDB** | CRDT-based multi-region conflict resolution. |
| **CRDT.tech / CRDT zoo** | Reference list of CRDT types. |
| **OrbitDB** | IPFS-based CRDT database. |
| **Soundgarden / SoundCloud** (per blog posts) | Internal CRDT usage. |
| **Akka Distributed Data** | CRDTs in the Akka ecosystem. |
| **Roshi (SoundCloud)** | LWW-element-set over Redis. |
| **AntidoteDB** | Research / open-source CRDT database. |
| **Apollo (Facebook)** | CRDT-based distributed cache. |
| **Logux** | CRDT-style client-server sync. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Figma** | CRDTs for offline mode and multiplayer document state. | ✅ Verified — Figma engineering blog |
| **Linear** | Yjs for offline-first collaborative project management. | ✅ Verified — Linear engineering blog |
| **Notion** | CRDT-based offline editing (per public talks). | ⚠ Discussed publicly; specifics vary |
| **Apple** | CRDB for collaborative editing in Notes (per WWDC talks). | ✅ Verified — Apple WWDC sessions |
| **Microsoft Azure** | CosmosDB conflict resolution. | ✅ Verified — Azure docs |
| **Riak (Basho)** | Native CRDTs since Riak 2.0. | ✅ Verified — Riak documentation |
| **Redis Labs** | Active-Active Redis Enterprise. | ✅ Verified — Redis Enterprise docs |
| **Mattermost** | Yjs for collaborative messaging features. | ✅ Verified — Mattermost engineering blog |
| **TomTom** | Geo-distributed routing data. | ✅ Verified — public talks |
| **SoundCloud** | Roshi (LWW-element-set over Redis). | ✅ Verified — Roshi open-source |
| **Tldraw, Excalidraw, Cursor, many editors** | Yjs in many real-time tools. | ✅ Common pattern |

---

## Further reading

- Shapiro et al., *Conflict-free Replicated Data Types* (SSS 2011) — the formal foundation.
- Shapiro et al., *A comprehensive study of CRDTs* (INRIA TR 2011) — the full survey.
- *Designing Data-Intensive Applications* (Kleppmann), Ch 5 on replication.
- Kleppmann's papers on CRDTs (Cambridge group) — many practical contributions.
- Yjs documentation — practical, performance-oriented.
- Automerge documentation — academically-influenced.
- Martin Kleppmann's *Local-First Software* essay — broader context.
- *CRDT for Mortals* by James Long — accessible introduction.
- *Conclave* (collaborative editor) — open-source teaching example.
- Riak documentation on data types.

---

*Diagram sources: [`../diagrams/src/crdt-concept.d2`](../diagrams/src/crdt-concept.d2), [`../diagrams/src/crdt-catalog.d2`](../diagrams/src/crdt-catalog.d2).*
