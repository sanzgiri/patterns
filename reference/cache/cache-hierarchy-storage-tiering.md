# Cache Hierarchy & Multi-Tier Storage

**Aliases:** Cache Hierarchy, Memory Hierarchy, Multi-Level Cache, L1/L2/L3 Caching, Tiered Caching; Multi-Tier Storage, Storage Tiering, Hot/Warm/Cold Storage, Information Lifecycle Management (ILM), Data Temperature
**Category:** Caching / Data / Scalability
**Sources:**
[Hennessy & Patterson — *Computer Architecture: A Quantitative Approach* (6th ed., 2017)](https://www.elsevier.com/books/computer-architecture/hennessy/978-0-12-811905-1) — canonical memory hierarchy reference ·
[Jeff Dean — *Numbers Every Programmer Should Know*](http://highscalability.com/numbers-everyone-should-know) ·
[Peter Norvig — *Teach Yourself Programming in Ten Years* (latency numbers)](https://norvig.com/21-days.html) ·
[AWS S3 Storage Classes documentation](https://aws.amazon.com/s3/storage-classes/) ·
[Azure Blob Storage access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview) ·
[GCP Cloud Storage classes](https://cloud.google.com/storage/docs/storage-classes) ·
[*Designing Data-Intensive Applications* (Kleppmann, 2017)](https://dataintensive.net/) — Ch 3 ·
[*Database Internals* (Petrov, O'Reilly 2019)](https://www.databass.dev/)

---

## Problem

Two patterns about the same fundamental insight: **the working set is much smaller than the total data**, and access patterns are heavily skewed (small fraction of data accounts for the bulk of accesses, often called the 80/20 or Zipf distribution). When that's true, you can give hot data fast, expensive storage and cold data slow, cheap storage — and the average user experience matches the hot tier while the bill matches the cold tier.

**Problem 1 (Cache Hierarchy)**: a single storage layer can't be both fast and big and cheap — physics and economics forbid it. SRAM is fast but tiny and expensive; DRAM is fast and moderate but volatile and bigger; SSD is fast and big; HDD is slow and very big; tape is glacial and basically free. The same trade-off plays out at every scale, from CPU caches to global CDNs. The pattern is to stack tiers: every layer caches its slower neighbor, with hot data promoted upward and cold data demoted downward.

**Problem 2 (Multi-Tier Storage)**: applied at the data-store layer specifically. A user uploads a file. Yesterday's file is accessed constantly; last year's file maybe once a decade for compliance. Charging the same per-GB rate for both is wasteful. Storage tiering moves data automatically to cheaper, slower tiers as it ages or based on access frequency. The same blob lives in different physical media as its temperature changes, transparently to most reads.

Both patterns rest on:
- **Skewed access patterns** (the working set is small).
- **Cost-latency-capacity trade-offs in storage media** (no single medium is best at everything).
- **Temporal locality** (recently accessed data likely to be accessed again).
- **Spatial locality** (nearby data likely to be accessed together — applies at hardware level).

> [!TIP]
> **ELI5 (Cache Hierarchy).** Your laptop has a tiny, super-fast scratch space (CPU cache), a bigger fast space (RAM), and a huge slow space (disk). The CPU constantly moves the things you're using right now into the fast space. The same idea repeats at every scale: your browser caches files locally; the CDN caches them at the edge; Redis caches DB rows in memory near your servers; the DB caches disk pages in RAM; the disk caches sectors in its buffer. Every layer hides the slowness of the next.

> [!TIP]
> **ELI5 (Multi-Tier Storage).** Cloud object stores (S3, Azure Blob, GCS) let you pick how fast and how expensive each blob is. New uploads go to fast/expensive storage. After 30 days, automatically move them to cheaper "infrequent access" storage. After a year, move them to "archive" storage that costs almost nothing but takes minutes to retrieve. Same files, different prices, transparent to your app.

## How the Cache Hierarchy works

> [!TIP]
> **ELI5.** Every storage tier in your system caches the slower one. Each tier is ~10× faster but ~10× smaller and more expensive. Effective latency depends mostly on the hit rate of the upper tier. Getting hit rate from 90% to 99% matters more than making the cache itself faster.

### The hierarchy

The full memory hierarchy spans 9+ orders of magnitude in latency:

![Cache hierarchy with latency numbers](../diagrams/svg/cache-hierarchy.svg)

A condensed version of the famous *Latency Numbers Every Programmer Should Know* (Dean / Norvig):

| Tier | Typical latency | Capacity |
|---|---|---|
| CPU register | 0.3 ns | tens of bytes |
| L1 cache | 1 ns | KBs per core |
| L2 cache | 4 ns | MBs per core |
| L3 cache | 10 ns | MBs shared |
| RAM (DDR5) | 100 ns | GBs |
| Local NVMe | 100 µs | TBs |
| Same-DC SSD | 0.5 ms | PBs |
| Redis / Memcached (LAN) | 0.5–2 ms | GBs–TBs |
| RDBMS query | 1–10 ms | TBs |
| CDN edge (global) | 5–50 ms | PBs |
| Object store (S3) | 30–100 ms | exabytes |
| Cross-region | 80–200 ms | unlimited |
| Archive (Glacier) | minutes–hours | unlimited |

Each step is ~10–100× slower and ~10–100× cheaper per byte. The *cost per byte × access frequency* product is what you actually pay.

### Why hierarchies work

Three properties make hierarchies effective:

1. **Skewed access (working set << total set)**: in most workloads, a small fraction of data accounts for most accesses. Wikipedia: a few percent of articles get most traffic. E-commerce: a few percent of products account for most page views. Database: most queries hit recent or popular rows. This is sometimes called the "80/20 rule" or Zipf distribution.

2. **Temporal locality**: data accessed recently is likely to be accessed again soon. User browsing a product page just viewed it; will probably view related products too.

3. **Spatial locality**: nearby data is likely to be accessed together. CPU loads a cache line (64 bytes); programs almost always need neighboring bytes too.

If access were uniform (every byte equally likely), caching would barely help. The skew is what gives the hierarchy its power.

### The math of hit rate

Effective latency at a cached tier:

```
effective_latency = hit_rate × cache_latency + (1 − hit_rate) × backing_latency
```

Worked example with Redis (1ms) in front of RDBMS (10ms):

| Hit rate | Effective latency |
|---|---|
| 80% | 0.8 × 1 + 0.2 × 10 = 2.8 ms |
| 90% | 0.9 × 1 + 0.1 × 10 = 1.9 ms |
| 95% | 0.95 × 1 + 0.05 × 10 = 1.45 ms |
| 99% | 0.99 × 1 + 0.01 × 10 = 1.09 ms |
| 99.9% | 0.999 × 1 + 0.001 × 10 = 1.009 ms |

The takeaway: **at high hit rates, additional hit-rate gains matter more than cache speed**. Going from 95% to 99% saves more than swapping Redis for a faster cache.

Conversely, at low hit rates, you're mostly paying backing-store latency anyway — the cache adds an extra hop with little benefit. Bad caches are worse than no caches.

### Patterns applied at every tier

The same caching patterns recur at every level of the hierarchy:

- **Cache-aside** ([page](cache-aside.md)): lazy fill on miss.
- **Read-through / Write-through / Write-behind** ([page](cache-managed-strategies.md)): different consistency/performance trade-offs.
- **TTL + LRU eviction**: bound staleness and footprint.
- **Cache invalidation** on writes (Phil Karlton's "one of the two hard problems").
- **Stampede protection** ([thundering herd](../res/thundering-herd.md)): coalesce concurrent misses.
- **Negative caching**: remember "not found" too, to avoid pounding the backing store.
- **Cache warming**: pre-populate before traffic.
- **Probabilistic admission**: don't promote one-hit wonders (TinyLFU; segmented LRU).

### Hierarchy composition in practice

A real web app stack:

1. Browser cache (disk; per-user).
2. CDN edge cache (global PoPs; per-region).
3. Application-server local cache (in-process LRU; per-instance).
4. Distributed cache (Redis / Memcached; per-cluster).
5. Database buffer pool (DB's own RAM cache of disk pages).
6. SSD-backed storage.
7. Cross-region replica (for DR / read-locality).
8. Object store (for blobs / cold data).
9. Archive (for compliance / DR).

Each layer absorbs traffic from those below. A request that only ever needs Tier 2 (CDN) never burdens Tiers 4–9. A 95% CDN hit rate means your app servers see 5% of traffic.

### Anti-patterns

- **Caching everything**: small/cheap items add cache overhead without helping.
- **No eviction strategy**: cache grows until it OOMs.
- **No invalidation**: stale data persists forever.
- **One-hit wonder pollution**: rare items evict popular ones.
- **Cache as source of truth**: data only in cache → loss on flush.
- **Ignoring stampedes**: cache miss + 1000 concurrent users = DB death.
- **Caching pre-personalized content** at a shared layer: serves wrong user data.
- **Long TTLs on mutable data**: stale UX.

## How Multi-Tier Storage works

> [!TIP]
> **ELI5.** Cloud storage has a "fast and expensive" mode, a "medium" mode, a "cheap but slow to read" mode, and a "near-free but takes hours" mode. Set a rule like "after 30 days, demote to cheap; after a year, demote to archive." Your app keeps reading from the same bucket; the system moves bytes around physical media as their access pattern changes.

### Tiers in cloud object stores

The picture, with AWS as the example (Azure and GCP have direct equivalents):

![Multi-tier storage](../diagrams/svg/multi-tier-storage.svg)

| Tier | AWS | Azure | GCP | Latency | Cost notes |
|---|---|---|---|---|---|
| **Hot** | S3 Standard | Blob Hot | Standard | First byte ms | Highest storage, no retrieval fee |
| **Cool** | S3 Standard-IA, One Zone-IA | Blob Cool | Nearline | Same ms | Lower storage; per-GB retrieval fee; 30-day min |
| **Cold** | S3 Glacier Instant Retrieval | Blob Cold | Coldline | Same ms | Even lower storage; higher retrieval; 90-day min |
| **Archive** | S3 Glacier Flexible / Deep Archive | Blob Archive | Archive | Minutes to hours retrieval | Cheapest storage; high retrieval cost + retrieval time; 90–180 day min |
| **Intelligent** | S3 Intelligent-Tiering | Lifecycle policies | Autoclass | Automatic | Auto-moves between tiers based on access |

Concrete US-East-1 pricing (early 2024, illustrative):
- S3 Standard: $0.023/GB/month
- S3 Standard-IA: $0.0125/GB/month (~55% cheaper) + $0.01/GB retrieval
- S3 Glacier Flexible: $0.0036/GB/month (~85% cheaper) + retrieval cost + retrieval delay
- S3 Glacier Deep Archive: $0.00099/GB/month (~95% cheaper) + slowest retrieval

For a petabyte of data accessed rarely, the difference between Standard and Deep Archive is **~$23,000/month vs $1,000/month**. At that scale, tiering pays for itself instantly.

### Tiering policies

Lifecycle / tiering rules:

- **Age-based**: "objects created >30 days ago → IA; >90 days → Glacier; >365 → Deep Archive."
- **Access-based**: AWS S3 Intelligent-Tiering monitors access patterns and auto-promotes/demotes.
- **Prefix-based**: different bucket prefixes get different default storage classes (`/hot/`, `/archive/`).
- **App-driven**: application API call explicitly transitions blobs (closed ticket → archive).
- **Tag-based**: blobs tagged for retention move on a schedule.

### Beyond cloud object stores

The pattern recurs:

- **Database storage tiering**: hot tables on NVMe; cold tables on HDD; very cold partitions exported to S3 (Snowflake, BigQuery have native tiering).
- **Time-series databases**: hot recent data in memory; older in compressed columnar; very old in object storage (InfluxDB Cloud, TimescaleDB, Prometheus + Thanos).
- **Search indexes**: Elasticsearch hot/warm/cold/frozen tiers; recent indices in RAM, older on SSD, archived on object store.
- **Log analytics**: recent logs in fast store; older in cheap archives (Datadog, Splunk SmartStore, Cribl tiering).
- **Email archives**: Office 365, Gmail tier old mail to cheaper storage.

### Pitfalls

**Retrieval cost surprises**: cool / archive tiers charge per-GB retrieval. If "rarely accessed" data turns out to be accessed often, the bill explodes. S3 Standard-IA: $0.01/GB retrieval; download 1TB once → $10. Sounds cheap until you do it daily.

**Latency surprises**: "cool" tiers don't have higher per-byte latency once data is available — they're slower because of *restore time* from archive. Glacier Flexible can take 1–12 hours; Deep Archive 12–48 hours. Plan UX around this ("your file is being restored; we'll notify you").

**Minimum storage durations**: cool tiers have minimum billing periods (S3 IA = 30 days; Glacier = 90 days; Deep Archive = 180 days). Putting data in and pulling it back out the next day pays for the full minimum.

**Metadata still hot**: object listing and search need a warm metadata index. The blobs can be cold, but the catalog must be fast.

**Lifecycle bugs**: a misconfigured policy can archive hot data. Recovery from archive is slow. Always preview lifecycle policies before applying.

**Mixing tiers in one logical object**: large objects can't be partially archived. Multi-part objects are tiered as a unit.

### Combining cache hierarchy with storage tiering

A complete data path for, say, a media platform like Spotify or YouTube:

1. **CDN edge** caches popular tracks/videos. 95%+ hit rate for top-of-tail content.
2. **Origin app server cache** (Redis) for moderately popular content not in CDN.
3. **Hot object storage** (S3 Standard) for full library, accessed by origin.
4. **Cool storage** (S3 IA) for old / unpopular content; CDN warms it on demand.
5. **Archive** for the long tail, takedown copies, raw masters.
6. **Database** for metadata, user activity (heavily cached at app server tier).
7. **Analytics warehouse** for log data with its own tiering.

The combination of caching and storage tiering means that the typical request touches the fastest tier, while the long-tail request touches successively slower (cheaper) tiers. The bill matches the cold tier; the user experience matches the hot tier.

### Trade-offs

Advantages:
- **Cost savings** — often 50–95% at scale.
- **Transparent to most code** — same API, different physical backing.
- **Scales to any size** — archive is essentially unlimited.
- **Standard cloud feature** — no DIY infrastructure.

Disadvantages:
- **Retrieval cost gotchas** can wipe out savings.
- **Restore delays** for archived data require UX changes.
- **Minimum storage durations** can backfire on churning data.
- **Lifecycle policy mistakes** can be expensive (archive of hot data).
- **Visibility into tier distribution** requires monitoring tooling.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Cache Hierarchy (Hennessy & Patterson)** | The canonical memory hierarchy. |
| **L1/L2/L3 CPU caches** | Hardware version. |
| **Browser cache** | Per-user tier. |
| **CDN** ([page](../scale/cdn.md)) | Global edge tier. |
| **Distributed cache** | Redis/Memcached tier. |
| **DB buffer pool** | DB's own RAM cache. |
| **[Cache-aside](cache-aside.md), [Read/Write-through](cache-managed-strategies.md)** | Strategies within each tier. |
| **S3 Intelligent-Tiering** | Automatic cloud tiering. |
| **Information Lifecycle Management (ILM)** | Enterprise umbrella term. |
| **Hot/warm/cold tiers in Elasticsearch, Splunk** | Search/log tiering. |
| **Glacier / Archive tiers** | Slowest, cheapest. |
| **[Polyglot Persistence](../data/polyglot-persistence.md)** | Multiple stores, often with implicit tiering. |
| **[Materialized Views](../data/materialized-view-index.md)** | Cache-by-precomputation tier. |
| **[Probabilistic Sketches](../data/probabilistic-sketches.md)** | "Lossy but tiny" tier for analytics. |

## When NOT to use

**Cache hierarchy**:
- Uniform access patterns (caching won't help).
- Tight latency budgets where cache miss is unacceptable (use sync replication instead).
- Tiny datasets that fit in memory anyway.
- Single-write, never-read workloads.

**Multi-tier storage**:
- Small datasets where tier savings are negligible.
- Highly unpredictable access where retrieval costs eat savings.
- Data that *must* be retrievable instantly (compliance scenarios where minutes matter).
- Small numbers of large objects where management overhead exceeds savings.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **CPU caches** | L1/L2/L3 on every modern chip |
| **Browser caches** | Chrome, Safari, Firefox |
| **CDN: CloudFront, Cloudflare, Fastly, Akamai, Azure CDN** | Edge tier |
| **Redis, Memcached, Hazelcast, Apache Ignite** | Distributed cache |
| **Caffeine, Guava Cache, Ristretto** | In-process LRU caches |
| **AWS S3 Standard, IA, Glacier, Deep Archive** | Storage tiers |
| **Azure Blob Hot, Cool, Cold, Archive** | Storage tiers |
| **GCP Standard, Nearline, Coldline, Archive** | Storage tiers |
| **Elasticsearch hot/warm/cold/frozen** | Search index tiering |
| **InfluxDB / Prometheus + Thanos / VictoriaMetrics** | Time-series tiering |
| **Snowflake, BigQuery, Redshift Spectrum** | Warehouse tiering |
| **Cribl, Splunk SmartStore** | Log tiering |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Every modern CPU** | L1/L2/L3 cache hierarchy. | ✅ Universal hardware design |
| **Netflix Open Connect** | Massive CDN tier of cache hierarchy. | ✅ Verified — Netflix Tech Blog |
| **Spotify** | CDN + per-region Redis + object store for tracks. | ✅ Verified — Spotify Engineering posts |
| **YouTube / Facebook video** | CDN + multi-tier storage for video libraries. | ⚠ Architecture inferred from talks |
| **Amazon (Prime Video, retail)** | Storage tiering canonical example. | ✅ AWS case studies |
| **Microsoft 365 / OneDrive** | Heavy Azure Blob tiering. | ✅ Microsoft documentation |
| **Dropbox** | Multi-tier storage; warm in Magic Pocket, cold in S3. | ✅ Verified — Dropbox Tech Blog |
| **Pinterest** | Multi-region S3 + tiered caching. | ✅ Pinterest engineering posts |
| **Datadog, Splunk, Datadog** | Log tiering for retention. | ✅ Vendor documentation |
| **Most large data platforms** | Tiered storage for warehouses. | ✅ Industry universal |

---

## Further reading

- *Computer Architecture: A Quantitative Approach* (Hennessy & Patterson, 6th ed.) — Ch 2.
- *Designing Data-Intensive Applications* (Kleppmann, 2017) — Ch 3.
- *Database Internals* (Petrov, 2019).
- Jeff Dean — *Numbers Every Programmer Should Know*.
- Peter Norvig — *Teach Yourself Programming in Ten Years* (latency numbers).
- AWS S3 storage classes documentation and case studies.
- Azure Blob access tiers documentation.
- GCP storage classes documentation.
- Netflix Open Connect — *How Netflix Delivers Video*.
- *Cache-Conscious Algorithms* (Bender et al., literature).
- Elasticsearch hot-warm-cold architecture guides.

---

*Diagram sources: [`../diagrams/src/cache-hierarchy.d2`](../diagrams/src/cache-hierarchy.d2), [`../diagrams/src/multi-tier-storage.d2`](../diagrams/src/multi-tier-storage.d2).*
