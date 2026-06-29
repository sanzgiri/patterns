# System Design Patterns Reference

A working reference covering the most-cited system design patterns, drawn from eight verified pattern catalogs (see `.knowledge/semantic/system-design-patterns-unified-registry.json` for the full 142-pattern registry).

Each page follows the same structure:
- **ELI5 callouts** on Problem and Solution
- **Inline D2-rendered diagrams** explaining components and state transitions
- **Variants & related patterns** with cross-links
- **When NOT to use**
- **Real-world implementations** (libraries, products)
- **Companies using it** — every claim marked ✅ verified by fetch or ⚠ widely-known-but-unverified

> If you're new here, start with the **[Themes that emerged across phases](#themes-that-emerged-across-phases)** below — it maps the families of patterns together. Then jump into any pattern that interests you; every page is self-contained but cross-linked to related patterns.

---

## Coverage status

| Phase | Scope | Patterns | Pages | Status |
|---|---|---|---|---|
| **A** | Most-cited patterns (≥3 sources) | 15 | 15 | ✅ complete |
| **B** | Joshi distributed-systems core | 20 | 20 | ✅ complete |
| **C** | Microservices / communication / data | 21 | 21 | ✅ complete |
| **D** | Caching, scaling, data structures, resilience, storage | 20 | 13 | ✅ complete |
| **E** | Ops & observability | ~20 | 8 | ✅ complete |
| **F** | Security + remaining architectural styles | ~10 | 4 | ✅ complete |
| **G** | Final additions (SBA, Edge, identity adjuncts, static stability, hierarchies, consolidation) | ~10 | 5 | ✅ complete |

**Total:** **86 pages · 170 D2 diagrams · ~165,000 words.**

---

### Architectural style (`arch/`)
- [Monolithic Architecture](arch/monolith.md)
- [Microservices Architecture](arch/microservices.md)
- [Service Decomposition (by Business Capability / Subdomain)](arch/service-decomposition.md)
- [Strangler Fig](arch/strangler-fig.md)
- [Anti-Corruption Layer](arch/anti-corruption-layer.md)
- [Service Template](arch/service-template.md)
- [Microservice Chassis](arch/microservice-chassis.md)
- [Claim Check & Static Content Hosting](arch/claim-check-static-hosting.md)
- [Space-Based Architecture & Edge Computing](arch/space-based-edge.md)
- [Compute Resource Consolidation](arch/compute-resource-consolidation.md)

### Data (`data/`)
- [CQRS — Command Query Responsibility Segregation](data/cqrs.md)
- [Event Sourcing](data/event-sourcing.md)
- [Leader-Follower Replication](data/leader-follower-replication.md)
- [Saga](data/saga.md)
- [Sharding (Horizontal Partitioning)](data/sharding.md)
- [Database per Service](data/database-per-service.md)
- [Replicated Log](data/replicated-log.md)
- [Write-Ahead Log (WAL)](data/wal.md)
- [Segmented Log](data/segmented-log.md)
- [B-Tree](data/btree.md)
- [LSM-Tree](data/lsm-tree.md)
- [MVCC — Multi-Version Concurrency Control](data/mvcc.md)
- [Transactional Outbox](data/outbox.md)
- [Change Data Capture (CDC)](data/cdc.md)
- [Consistent Hashing](data/consistent-hashing.md)
- [Range Partitioning & Geohash](data/range-partitioning-geohash.md)
- [Probabilistic Sketches (Bloom, HLL, Count-Min)](data/probabilistic-sketches.md)
- [Merkle Tree](data/merkle-tree.md)
- [CRDT (Conflict-Free Replicated Data Type)](data/crdt.md)
- [Materialized View & Index Table](data/materialized-view-index.md)
- [Inverted Index](data/inverted-index.md)
- [Polyglot Persistence (& Storage Taxonomy)](data/polyglot-persistence.md)

### Caching (`cache/`)
- [Cache-Aside (Lazy Loading)](cache/cache-aside.md)
- [Read-Through / Write-Through / Write-Behind (Cache-Managed Strategies)](cache/cache-managed-strategies.md)
- [Cache Hierarchy & Multi-Tier Storage](cache/cache-hierarchy-storage-tiering.md)

### Communication (`comm/`)
- [API Gateway](comm/api-gateway.md)
- [Publish-Subscribe](comm/pub-sub.md)
- [Backends for Frontends (BFF)](comm/bff.md)
- [Sidecar](comm/sidecar.md)
- [Service Mesh](comm/service-mesh.md)
- [Ambassador](comm/ambassador.md)
- [Service Registry](comm/service-registry.md)
- [Client-Side Service Discovery](comm/client-side-discovery.md)
- [Server-Side Service Discovery](comm/server-side-discovery.md)
- [State Watch](comm/state-watch.md)
- [Request Batching & Pipelining](comm/batching-pipelining.md)
- [Idempotent Consumer](comm/idempotent-consumer.md)
- [API Composition](comm/api-composition.md)
- [GraphQL Federation](comm/graphql-federation.md)
- [Consumer-Driven Contract Test](comm/consumer-driven-contract-test.md)
- [Service Component Test](comm/service-component-test.md)

### Resilience (`res/`)
- [Circuit Breaker](res/circuit-breaker.md)
- [Throttling / Rate Limiting](res/throttling.md)
- [Bulkhead](res/bulkhead.md)
- [Thundering Herd / Cache Stampede](res/thundering-herd.md)
- [Backpressure](res/backpressure.md)
- [Static Stability](res/static-stability.md)

### Scalability (`scale/`)
- [Content Delivery Network (CDN)](scale/cdn.md)

### Coordination (`coord/`)
- [Two-Phase Commit](coord/2pc.md)
- [Paxos](coord/paxos.md)
- [Raft](coord/raft.md)
- [Gossip Protocol](coord/gossip.md)
- [Consistent Core](coord/consistent-core.md)
- [Emergent Leader](coord/emergent-leader.md)

### Building blocks (`block/`)
- [Lamport Clock](block/lamport-clock.md)
- [Vector Clock / Version Vector](block/vector-clock.md)
- [Generation Clock](block/generation-clock.md)
- [Heartbeat](block/heartbeat.md)
- [High-Water Mark & Low-Water Mark](block/hwm-lwm.md)
- [Quorum](block/quorum.md)
- [Lease](block/lease.md)
- [Hybrid Logical Clock (HLC)](block/hybrid-logical-clock.md)
- [Clock-Bound Wait](block/clock-bound-wait.md)
- [Singular Update Queue](block/singular-update-queue.md)

### Operations / Observability (`ops/`)
- [Blue-Green & Canary Deployment (Progressive Delivery)](ops/blue-green-canary.md)
- [Feature Flag (Feature Toggle)](ops/feature-flag.md)
- [Deployment Stamps & Geode (Cellular and Geographic Distribution)](ops/deployment-stamps.md)
- [Externalized Configuration](ops/externalized-config.md)
- [Distributed Tracing & Correlation ID](ops/distributed-tracing.md)
- [Log Aggregation & Application Metrics](ops/log-aggregation-metrics.md)
- [Chaos Engineering](ops/chaos-engineering.md)
- [Audit Logging](ops/audit-log.md)

### Security (`sec/`)
- [Federated Identity (SSO / OAuth / OIDC / SAML)](sec/federated-identity.md)
- [Gatekeeper & Valet Key](sec/gatekeeper-valet-key.md)
- [SCIM, Passkeys (FIDO2/WebAuthn), and Kerberos](sec/scim-passkeys-kerberos.md)

---

## Themes that emerged across phases

- **Building blocks compose**: Lamport/vector/HLC clocks → heartbeat/lease/quorum → Paxos/Raft → consistent-core → service registry → service mesh. Each layer reuses primitives from below.
- **The dual-write trilogy**: Outbox + CDC + Idempotent Consumer solve "atomically update DB and emit event" — appears everywhere in microservices.
- **Cross-service queries**: API Composition / CQRS Read Model / GraphQL Federation are the three answers to "how do I show data from multiple services?"
- **Test strategy pyramid**: Contract tests (catch breaking changes early) + Component tests (verify service in isolation) + few E2E (smoke).
- **Platform team's toolbox**: Template + Chassis + Mesh + Sidecar + Discovery — the kit every platform team builds.
- **Distribution primitives**: Consistent hashing for point lookups, range for ordered scans, geohash for 2D — every sharded system uses some mix.
- **Probabilistic sketches**: Bloom, HLL, Count-Min Sketch are the modern "tiny memory, big stream" toolbox — used in observability/analytics at every scale.
- **Materialized views as a unifying concept**: search indexes, cached query results, denormalized tables, CQRS read models, stream-processed views — all "precompute at write time so reads are cheap."
- **The resilience trinity**: Bulkhead isolates failures + Circuit Breaker stops sending to broken things + Backpressure propagates load signals → prevents cascading-failure death spiral.
- **Progressive delivery family**: Blue-Green + Canary + Feature Flag + Shadow → deploy ≠ release; lets you decouple risk decisions from code changes.
- **Three pillars of observability**: Logs + Metrics + Traces, stitched together via correlation/trace IDs, are the modern debugging foundation.
- **Cell-based scaling**: stamps + geodes + shuffle sharding + AZ/region isolation compose into hyperscale architectures (Slack, AWS, Salesforce all converged here).
- **Event sourcing is the deep version**: events as source of truth + projections as derived state — pairs naturally with CQRS, Outbox, CDC, audit logs.
- **The two perimeter problems**: Gatekeeper protects the front door; Valet Key (presigned URLs) lets clients reach storage directly without proxying through your app.
- **Identity has three orthogonal layers**: SCIM (who exists), passkeys/WebAuthn (how they prove it), federated SSO (how that proof is accepted) — a modern enterprise needs all three.
- **The centralization escape hatches**: Space-Based Architecture removes the DB bottleneck by keeping hot state in an in-memory grid; Edge Computing removes the round-trip-to-region bottleneck by running code at PoPs near users. Both answer "the central tier is the wrong place."
- **Static Stability is the AWS-style resilience axiom**: don't react during failures; pre-provision so doing nothing keeps the system working.
- **Consolidation vs isolation is the eternal trade-off**: bin packing saves money; bulkheads prevent cascading failures. Modern stacks (Kubernetes, serverless) try to give you both.

---

## Diagram tooling

All diagrams are written in [D2](https://d2lang.com/) and rendered to SVG:

```bash
# Render a single diagram
d2 diagrams/src/circuit-breaker-flow.d2 diagrams/svg/circuit-breaker-flow.svg

# Render all
for f in diagrams/src/*.d2; do
  d2 "$f" "diagrams/svg/$(basename "$f" .d2).svg"
done
```

Source `.d2` files are committed in `diagrams/src/`; rendered SVGs in `diagrams/svg/`. Both are version-controlled so the rendering is reproducible and the diffs are reviewable.

---

## Sources behind the registry

| Source | URL | Scope |
|---|---|---|
| Microsoft Azure Cloud Design Patterns | https://learn.microsoft.com/en-us/azure/architecture/patterns/ | Cloud architectural patterns (43) |
| Chris Richardson — microservices.io | https://microservices.io/patterns/index.html | Microservice-specific (~50) |
| Unmesh Joshi — Patterns of Distributed Systems | https://martinfowler.com/articles/patterns-of-distributed-systems/ | Distributed-system implementation patterns (30) |
| Martin Kleppmann — DDIA (conceptual) | https://dataintensive.net/ | Data-systems textbook (not a numbered catalog) |
| Neo Kim — systemdesign.one | https://systemdesign.one/system-design-interview-cheatsheet/ | Interview / scaling cheatsheet |
| Alex Xu — ByteByteGo system-design-101 | https://github.com/ByteByteGoHq/system-design-101 | Visual / interview overview |
| Hohpe & Woolf — *Enterprise Integration Patterns* | https://www.enterpriseintegrationpatterns.com/ | Messaging patterns (Claim Check, Pub-Sub, etc.) |
| Fowler / Young / Vernon — DDD + Event Sourcing | https://martinfowler.com/eaaDev/EventSourcing.html | Event sourcing, CQRS, DDD |

All page citations are linked back to these primary sources. Per-pattern facts and company claims are flagged ✅ or ⚠ for verification status.
