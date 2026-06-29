# Space-Based Architecture & Edge Computing

**Aliases:** Space-Based Architecture (SBA), Tuple Space, In-Memory Data Grid (IMDG), Distributed Cache + Compute, Cloud-Scale Pattern; Edge Computing, Edge Compute, Multi-Access Edge Computing (MEC), Fog Computing, Edge AI
**Category:** Architecture / Scalability
**Sources:**
[Mark Richards — *Software Architecture Patterns* (O'Reilly 2015)](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/) — original named source for SBA ·
[Hazelcast — In-Memory Data Grid documentation](https://hazelcast.com/glossary/in-memory-data-grid/) ·
[Apache Ignite documentation](https://ignite.apache.org/docs/latest/) ·
[Linda / Tuple Space (Gelernter, 1985)](https://dl.acm.org/doi/10.1145/2363.2433) — academic origin ·
[ETSI Multi-access Edge Computing standards](https://www.etsi.org/technologies/multi-access-edge-computing) ·
[Cloudflare Workers documentation](https://developers.cloudflare.com/workers/) ·
[AWS Lambda@Edge documentation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html) ·
[Vercel Edge Functions documentation](https://vercel.com/docs/functions/edge-functions) ·
[NIST Fog Computing definition (SP 500-325)](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.500-325.pdf)

---

## Problem

Two complementary "the central server is the wrong place to do this" patterns from opposite ends of the scale spectrum:

**Space-Based Architecture (centralization makes the DB the bottleneck)**: In a classic web/app/DB stack you can scale the web and app tiers horizontally by adding servers — but eventually every request still goes to one database. Under sustained load or burst (flash sale, ticket release, market open) the DB saturates; app servers block waiting for connections; latencies cascade. Vertical-scaling the DB hits a ceiling and gets ruinously expensive.

**Edge Computing (latency to a central region is the wrong floor)**: A central cloud region in `us-east-1` is ~150ms from Singapore, ~80ms from London. For interactive UX, IoT, video, AR/VR, that's too slow. And bandwidth: streaming raw video from a million security cameras to the cloud is wasteful when a tiny ML model on the camera could pre-filter. And data sovereignty: GDPR / PIPL / industry rules demand certain data never leave a region or even a building.

Both patterns answer: **the central tier is the wrong place — move state and/or compute closer to the work**. SBA does this by moving state into a distributed in-memory grid co-located with compute units. Edge computing does it by moving compute out to CDN PoPs, telco base stations, or even the device itself.

> [!TIP]
> **ELI5 (Space-Based).** Instead of every app server fighting for one database, keep all the hot data in a shared "in-memory grid" — basically a giant distributed dictionary spread across many servers. Each server holds a slice in RAM. Business logic runs on the same server as the data, so reads/writes are local (microseconds, not milliseconds). A background process drains updates to the real database asynchronously. The DB is no longer in the hot path; you scale by adding more grid nodes.

> [!TIP]
> **ELI5 (Edge Computing).** Don't send every request to a faraway data center. Run code at servers near the user — at CDN edge locations (300+ worldwide), at the cell tower, at the retail store, on the device itself. Personalization, auth, rewrites, simple ML — all can happen at the edge in a few milliseconds instead of 150ms across an ocean. Plus you can keep working offline and use less bandwidth.

## How Space-Based Architecture works

> [!TIP]
> **ELI5.** Take the data out of the database and put it in shared RAM across all your app servers (a "data grid"). The grid figures out where each piece lives and replicates it for safety. Your code reads/writes the grid like a fast local cache. A separate writer process persists changes to disk in the background, so if the grid loses power you can recover. Result: you scaled past the DB ceiling by removing the DB from the hot path.

### Architecture

The pattern:

![Space-based architecture](../diagrams/svg/space-based-architecture.svg)

Components (from Richards' original formulation):

- **Processing Units (PUs)**: identical, ephemeral compute nodes. Each holds a partition of the data in-memory and runs the business logic that operates on it. PUs come and go; data rebalances via [consistent hashing](../data/consistent-hashing.md) (or similar).
- **The Space (data grid)**: a distributed in-memory store shared across PUs. Conceptually a tuple space (Linda, 1985); practically a partitioned + replicated KV store (Hazelcast, Apache Ignite, Coherence, GridGain, GemFire, Memcached cluster).
- **Messaging Grid**: routes external requests to the right PU (the one holding the relevant data partition).
- **Processing Grid**: orchestrates multi-PU workflows when a single request touches multiple partitions.
- **Deployment Grid**: manages PU lifecycle — start, stop, scale, failover.
- **Data Writer**: asynchronously drains updates from the space to the backing persistent store (RDBMS, object store, event log).
- **Data Reader**: lazily loads cold data from the persistent store into the space on demand.

### Why this beats traditional N-tier

In traditional N-tier:
- Every read and write goes to the central DB.
- Connection pools, query queues, lock contention bottleneck at the DB.
- Scaling out app servers makes the DB worse (more connections, more contention).

In space-based:
- Reads/writes hit local in-memory partitions on the PU itself.
- The "DB" is just an eventual-persistence backstop, not the source of truth for live operations.
- Scaling out means more PUs → more partitions → more aggregate throughput → linear-ish scaling.

The trade-off is that you've made eventual persistence the model: a power loss before drain costs you in-flight transactions. Real systems use replication (each partition lives on 2-3 PUs) and ordered logs to make this acceptable.

### Canonical real-world uses

- **Financial trading / market-making**: order books in-memory; latency is competitive advantage. NYSE, NASDAQ, many HFT firms.
- **Airline reservations / GDS**: Amadeus, Sabre — flight inventory in IMDGs.
- **Online betting**: peak load during major events; Bet365, DraftKings.
- **Risk calculation grids**: banks run overnight risk grids on data in IMDGs.
- **Large multiplayer games**: region servers hold player state in shared grids.
- **Telco BSS/OSS**: high-volume CDR processing.
- **E-commerce flash sales**: Singles Day-scale events.

### Trade-offs (SBA)

Advantages:
- **Removes the DB bottleneck** for the hot path.
- **Predictable latency** (microseconds in-memory).
- **Burst-tolerant**: spin up more PUs; rebalance.
- **Mature implementations**: Hazelcast, Ignite, Coherence, GemFire have been around 15+ years.

Disadvantages:
- **In-memory size limits**: TBs not PBs; can't store the whole web.
- **Eventual persistence**: data loss windows possible on grid failure.
- **Operational complexity**: split-brain, rebalancing, partition tuning.
- **Framework lock-in**: each grid has its own APIs/idioms.
- **Wrong for ad-hoc OLAP**: built for OLTP-style access, not big SQL.
- **Cost**: RAM is expensive; lots of nodes.
- **Not for documents / big blobs**: use object storage.

## How Edge Computing works

> [!TIP]
> **ELI5.** Imagine your website's "brain" is currently in one place — say Virginia. A user in Tokyo has to wait 150ms each time they ask a question. Edge computing puts mini-brains at 300+ locations worldwide. The Tokyo user now talks to a brain 5ms away in Tokyo for most things. For things only the Virginia brain can answer (like "what's in my account?"), the edge brain forwards the question — but everything else (personalization, auth, A/B test pick, header rewrites) happens locally.

### The spectrum

Edge isn't a single tier — it's a continuum:

![Edge computing spectrum](../diagrams/svg/edge-computing.svg)

From farthest to nearest:

| Tier | Latency to user | Examples |
|---|---|---|
| **Central cloud region** | 50–200ms global | AWS `us-east-1`, GCP `europe-west1` |
| **CDN PoP (cache only)** | 5–50ms | CloudFront, Cloudflare, Fastly, Akamai |
| **Edge compute (PoP)** | 5–50ms | Workers, Lambda@Edge, Vercel Edge, Fastly Compute |
| **MEC (telco edge)** | 1–10ms | AWS Wavelength, Azure Edge Zones (5G base stations) |
| **On-prem / store edge** | <1ms (LAN) | AWS Outposts, Azure Stack Edge |
| **Device / IoT** | 0ms (local) | Phone, sensor, car, drone |

Each tier has its own constraints: edge compute platforms typically have CPU/memory limits, restricted runtimes (V8 isolates rather than full containers), bounded execution time (<50ms typically), and KV-style state only (no SQL).

### What runs at the edge

**Good fit**:
- Personalization at request time (greet user by name; show region-specific content).
- A/B testing decisions (pick variant per request).
- Auth token validation, JWT verification.
- Geo-routing, header rewrites, URL normalization.
- IoT data filtering before upload ("only send anomalies").
- AR/VR / gaming low-latency hot path.
- Video/image transformation (resize, crop, format-convert).
- Small ML inference (intent classification, recommendation pre-filter).
- Bot detection, basic WAF rules.
- Response composition from multiple upstreams.

**Bad fit**:
- Large-state OLTP (data is centralized).
- Heavy compute (large ML training or inference).
- Complex SQL joins across global data.
- Cross-region distributed transactions.
- Anything needing strong consistency across edge nodes.

### Edge state

Edge platforms increasingly offer state, with caveats:

- **Cloudflare Workers KV**: eventually consistent global KV (seconds to converge).
- **Cloudflare D1**: SQLite at the edge (read replicas).
- **Cloudflare Durable Objects**: single-leader objects pinned to a region (strong consistency for that object).
- **Cloudflare R2**: S3-compatible storage at edge.
- **Vercel Edge Config**: read-mostly config at edge.
- **Vercel KV / Postgres**: hosted Redis / Postgres for edge functions.
- **DynamoDB Global Tables**: multi-region tables with last-writer-wins.
- **Fastly KV Store**: KV at edge.

The general rule: **eventually consistent at edge; strong consistency requires a central authority** (which means a round trip when you need it).

### Edge patterns

- **Edge Cache**: cache responses at edge (the classic CDN). See [CDN page](../scale/cdn.md).
- **Edge Compute**: run code at PoP for personalization and routing.
- **Edge State (eventually consistent)**: small KV per edge region.
- **Single-region Durable Objects**: state pinned to one PoP for strong consistency.
- **Offline-first**: local-first apps that sync when online (CouchDB/PouchDB; CRDT-based).
- **Progressive enhancement**: SSR at edge, hydrate on device.
- **[Geode pattern](../ops/deployment-stamps.md)**: geographic deployment stamps + global routing for major regions.
- **IoT tiering**: device → gateway → cloud (filter and aggregate at each layer).
- **Hybrid edge + origin**: edge handles common case; falls through to origin for the rest.

### Real platforms

| Platform | Runtime | State | Notes |
|---|---|---|---|
| **Cloudflare Workers** | V8 isolates (JS, Wasm) | KV, D1, R2, Durable Objects | Most-developed edge ecosystem |
| **AWS Lambda@Edge** | Node, Python | None at edge; falls back to region | CloudFront-integrated |
| **AWS CloudFront Functions** | JavaScript (tiny) | Stateless | Ultra-cheap, ultra-restricted |
| **AWS Wavelength** | EC2 / containers | Full AWS | Telco edge zones (5G) |
| **Vercel Edge Functions** | V8 isolates | Edge Config, KV | Tightly Next.js-integrated |
| **Fastly Compute@Edge** | Wasm (Rust, JS, Go) | KV Store | Wasm-focused |
| **Netlify Edge Functions** | Deno | Edge state coming | |
| **Deno Deploy** | Deno | KV | First-class Deno |
| **Akamai EdgeWorkers** | JavaScript | EdgeKV | Akamai-native |
| **Azure Front Door + Functions** | Various | Cosmos DB global | Microsoft stack |

### Trade-offs (Edge)

Advantages:
- **Low latency** (5–50ms vs 100–200ms).
- **Bandwidth savings** (filter at edge).
- **Resilience** (regional outage limited).
- **Data locality** (GDPR / sovereignty compliance).
- **Better UX** (faster first byte, faster interactions).
- **Cost** (often cheaper than central egress).

Disadvantages:
- **Restricted runtimes** (no full Linux usually).
- **Limited state** (eventually consistent at best for global writes).
- **Operational complexity** (more places to deploy and debug).
- **Vendor lock-in** (each platform has its own APIs).
- **Observability harder** (logs scattered across PoPs).
- **Not suitable for heavy compute** (CPU/memory caps).

### When SBA and Edge meet

Sophisticated systems combine them:
- Per-region SBA cluster (in-memory grid in each region).
- Edge compute does request routing, auth, personalization.
- DB tier is for archival / analytics.

This is essentially the modern shape of e-commerce, ad-serving, gaming, and many real-time systems at scale.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Space-Based Architecture** | Richards' formulation; IMDG-centric. |
| **Tuple Space (Linda)** | Academic ancestor. |
| **In-Memory Data Grid (IMDG)** | The storage layer of SBA. |
| **CQRS** ([page](../data/cqrs.md)) | Often combined: writes hit space, reads via projections. |
| **Event Sourcing** ([page](../data/event-sourcing.md)) | Often combined: events drain to log; space holds derived state. |
| **Sharding** ([page](../data/sharding.md)) | Space-based grid is essentially sharded RAM. |
| **Polyglot Persistence** ([page](../data/polyglot-persistence.md)) | SBA can be one tier of polyglot. |
| **Edge Computing** | The geographic-distribution complement. |
| **Fog Computing** | NIST term for compute between cloud and device. |
| **MEC (Multi-access Edge Computing)** | ETSI term for telco edge. |
| **CDN** ([page](../scale/cdn.md)) | Edge cache; subset of edge computing. |
| **[Geode pattern](../ops/deployment-stamps.md)** | Geographic stamps; cousin to edge. |
| **Edge AI / TinyML** | ML inference at device edge. |
| **Offline-first / local-first software** | Edge taken to device. |
| **[CRDT](../data/crdt.md)** | Conflict resolution for offline-first sync. |

## When NOT to use

**Space-Based:**
- DB is genuinely not the bottleneck (most apps).
- Data doesn't fit in RAM at any practical cost.
- Need ad-hoc OLAP, document search, complex SQL.
- Team lacks operational experience with IMDGs.
- Eventual persistence is unacceptable (regulated ledger).

**Edge Computing:**
- App is internal / low-latency-insensitive.
- Workload needs heavy state or compute.
- Most users are in one region anyway.
- Operational complexity exceeds latency benefit.
- Workload can't fit in V8/Wasm constraints.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **Hazelcast, Apache Ignite, GridGain, Oracle Coherence, GemFire** | IMDGs |
| **Redis cluster, Memcached** | Lighter IMDGs |
| **Cloudflare Workers / KV / D1 / R2 / Durable Objects** | Edge compute + state |
| **AWS Lambda@Edge, CloudFront Functions, Wavelength, Outposts** | AWS edge |
| **Azure Front Door + Functions, Azure Stack Edge** | Azure edge |
| **Vercel Edge Functions / Edge Config / KV** | Edge for Next.js |
| **Fastly Compute@Edge** | Wasm edge |
| **Netlify Edge Functions, Deno Deploy** | JS/TS edge |
| **Akamai EdgeWorkers / EdgeKV** | Akamai edge |
| **Core ML, TF Lite, ONNX Runtime, PyTorch Mobile** | Device ML inference |
| **AWS Greengrass, Azure IoT Edge** | Industrial IoT edge runtimes |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **NYSE, NASDAQ, major HFT firms** | In-memory order books (SBA-style). | ✅ Universal in financial trading |
| **Amadeus, Sabre** | Airline inventory in IMDGs. | ✅ Public case studies (Hazelcast, GridGain) |
| **Bet365** | Hazelcast-based architecture for peak load. | ✅ Hazelcast case study |
| **Walmart** | Major IMDG users for inventory. | ✅ Documented in talks |
| **Cloudflare** | Edge compute serves ~20% of web traffic. | ✅ Verified — Cloudflare blog |
| **Discord** | Heavy edge-region architecture. | ✅ Discord engineering blog |
| **Shopify** | Edge for personalization, A/B. | ✅ Verified — Shopify Engineering |
| **Netflix** | Edge caches (Open Connect) + region compute. | ✅ Verified — Netflix Tech Blog |
| **Tesla** | On-vehicle inference; edge OTA. | ✅ Industry-known |
| **Industrial IoT (GE, Siemens)** | Factory floor edge compute. | ✅ Industry standard |
| **Vercel customers (Next.js apps)** | Edge functions widely deployed. | ✅ Vercel docs / case studies |

---

## Further reading

- *Software Architecture Patterns* (Mark Richards, O'Reilly 2015) — original SBA chapter.
- Hazelcast / Apache Ignite / GemFire documentation.
- Gelernter — *Generative Communication in Linda* (1985) — academic root.
- *Designing Distributed Systems* (Burns, 2018) — patterns chapter.
- *Cloud Native Patterns* (Davis, Manning 2019).
- ETSI MEC specifications.
- NIST Fog Computing reference (SP 500-325).
- Cloudflare Workers, Vercel Edge, Fastly Compute — product docs and engineering blogs.
- *Local-First Software* (Kleppmann et al., 2019).
- *Edge AI* (Daniel Situnayake, O'Reilly 2023).

---

*Diagram sources: [`../diagrams/src/space-based-architecture.d2`](../diagrams/src/space-based-architecture.d2), [`../diagrams/src/edge-computing.d2`](../diagrams/src/edge-computing.d2).*
