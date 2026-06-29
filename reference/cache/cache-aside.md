# Cache-Aside (Lazy Loading)

**Aliases:** Lazy Loading, Application-Managed Cache, Look-Aside Cache, "Cache then DB"
**Category:** Caching
**Sources:**
[Microsoft Azure — Cache-Aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside) ·
[AWS — Database caching strategies (Cache-Aside)](https://aws.amazon.com/caching/best-practices/) ·
[Facebook (Meta) — *Scaling Memcache at Facebook*, NSDI 2013](https://research.facebook.com/publications/scaling-memcache-at-facebook/) ·
Discussion in [systemdesign.one](https://systemdesign.one/caching/) and ByteByteGo system-design-101

---

## Problem

> [!TIP]
> **ELI5.** Database reads are slow (10ms+ for a complex query); RAM reads are fast (sub-millisecond). You'd like to remember the answer to "what's user 42's profile?" so the next 1000 requests don't all hit the DB. A **cache** is a separate fast key-value store (Redis, Memcached) that holds copies of frequently-accessed data. The hard part isn't reading from a cache; it's keeping it correct when the underlying DB changes. **Cache-Aside** is the simplest and most common strategy: the *application* talks to both — checks the cache first, falls back to the DB on miss, populates the cache, and invalidates the cache when it writes.

The motivation is universal:

- **Reads dominate writes**: a typical user-profile page might be read 10,000 times for each time it's written.
- **Read latency matters**: every 100ms of latency costs measurable revenue at scale (Amazon, Google have published numbers).
- **DBs are expensive to scale**: another read replica costs real money; another GB of Redis is cheap.
- **Most data doesn't change often**: the user's name, the product's title, the article's content — minutes-to-hours staleness is acceptable.

A cache solves all four. But once you have a cache, you have a new problem: **two copies of the truth** that can disagree. Cache-aside is one answer (the most common). The alternatives — read-through, write-through, write-behind (see [Cache-Managed Strategies](cache-managed-strategies.md)) — make different trade-offs around who manages the consistency.

Cache-aside is dominant because:
- It works with any plain key-value store; the cache doesn't need to know about your DB.
- The application has full control over what to cache, when, and how.
- Cache failures don't break the application — they just make it slower.

The cost: every piece of application code that reads data has to know about the cache. And the consistency model is "eventually consistent with a small race window," which is *usually* fine but occasionally surprising.

## How it works

> [!TIP]
> **ELI5.** Read path: ask the cache; if it has the answer (HIT), return it. If not (MISS), ask the DB, then put the answer in the cache for next time. Write path: update the DB, then *delete* the cache entry (don't try to update it — just throw it away; the next read will refresh).

The two flows are simple and asymmetric:

![Cache-Aside read and write flows](../diagrams/svg/cache-aside-flow.svg)

**Read path (lazy population):**

```python
def get_user(user_id):
    # 1. Try the cache first
    cached = cache.get(f"user:{user_id}")
    if cached is not None:
        return cached                       # HIT: done, fast

    # 2. MISS: fall back to DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)

    # 3. Populate cache for next time (with TTL)
    cache.set(f"user:{user_id}", user, ttl=300)  # 5 min
    return user
```

**Write path (invalidate, don't update):**

```python
def update_user(user_id, changes):
    # 1. Update the source of truth first
    db.execute("UPDATE users SET ... WHERE id = ?", changes, user_id)

    # 2. Invalidate the cache entry (delete, not update)
    cache.delete(f"user:{user_id}")
```

The asymmetry is intentional. On read, the cache is *optional* (it's an optimization). On write, the cache *must* be made consistent — and the safest way to do that is to delete the entry, forcing the next reader to refresh from the source of truth.

### Why invalidate, not update?

A common beginner instinct on the write path is "update the cache with the new value." This is wrong:

- **The DB might reject the write.** If you update cache then DB, and the DB fails, the cache has a value that never made it to the source of truth.
- **The DB might write the value differently than your cache expects.** Triggers, defaulted columns, type coercion — what's in the DB is not always what you sent.
- **Cache updates and DB updates can race with other writers.** Writer A updates DB to v2 and cache to v2; Writer B updates DB to v3 and cache to v3 — but in different order. Cache might end up with v2 while DB has v3.

Deletion is **idempotent** and **safe**: the worst case is one extra DB read by the next reader. That's a tiny price for a much simpler consistency story.

### TTL: the safety net

Every cached entry should have a **TTL (time-to-live)** — an expiration. TTLs serve as:

- **A safety net against missed invalidations.** If a write somehow forgets to invalidate (a bug, a crash between DB write and cache delete), the cache will self-heal when the TTL fires. The wrongness is bounded.
- **A memory-management tool.** Without TTLs, the cache fills up with everything ever read, including values for objects that no longer exist.
- **A consistency lever.** Shorter TTL = fresher data but more cache misses. Longer TTL = better hit rate but more potential staleness.

Typical TTL choices:
- **Seconds** for highly dynamic data (stock prices, inventory counts).
- **Minutes** for user profiles, product details.
- **Hours** for relatively static reference data (country codes, currency conversions).
- **Days** for nearly-immutable data (with explicit invalidation on the rare change).

### The four pitfalls

Cache-aside is simple but has classic failure modes worth knowing:

![Cache-Aside pitfalls](../diagrams/svg/cache-aside-pitfalls.svg)

**1. Stale-read race.** This is subtle and famous:

```
T0: ReaderA: GET key       → MISS
T1: ReaderA: SELECT row    → returns v1 (in flight)
T2: Writer:  UPDATE to v2; DELETE cache key
T3: ReaderA: SET key=v1    ← writes stale value!
```

ReaderA's `SET` arrived *after* the writer's `DELETE`, so the cache ends up with v1 even though the DB has v2. The cache stays wrong until the TTL fires.

Mitigations:
- **Bounded TTL** — caps the damage in time.
- **Versioned keys** — include a version in the key (`user:42:v1234`); writes increment the version, leaving the old entry to expire on TTL.
- **Compare-and-swap** — only `SET` if the cache key is still empty (Redis `SET NX`).
- **CDC-driven invalidation** — invalidate from the database changelog rather than from application code, so invalidation happens *after* the DB write commits.
- **Switch to write-through** — eliminates the window entirely (at the cost of write latency).

**2. Cache stampede (thundering herd).** A hot key's TTL expires. Within milliseconds, 1000 concurrent requests all MISS, all hit the DB. The DB collapses.

Mitigations:
- **Single-flight / request coalescing**: when a MISS happens, only the *first* request fetches from DB; others wait for that result. Libraries: Go's `singleflight`, Java's `Caffeine` with `CacheLoader`, Python's `aiocache`.
- **Probabilistic early expiration**: each reader rolls a dice as TTL approaches; one of them refreshes early before the actual expiry.
- **Cache locks**: explicit lock on the key during refresh.
- **Background refresh**: a job preemptively refreshes hot keys before they expire.

Facebook's NSDI paper covers this in depth — it's a real problem at scale.

**3. Negative caching / key enumeration.** An attacker (or buggy client) requests 1 million random IDs that don't exist. Each MISSes the cache and hits the DB. The DB is overwhelmed by useless queries.

Mitigations:
- **Cache "not found"** as well as found values (with a shorter TTL like 30s).
- **Bloom filter** as a front line: if the filter says "definitely not in the DB," skip the DB query entirely. Pinterest, Discord, and others use this.
- **Rate-limit per-IP enumeration** at the gateway.

**4. Dual-write inconsistency.** The classic problem: between `UPDATE DB` and `DELETE cache`, the process crashes. The DB has the new value; the cache holds the old. Until the TTL fires (could be hours), readers see stale data.

Mitigations:
- **Accept it within TTL bounds** — bound the damage and move on. Most pragmatic.
- **Use [CDC](../data/cdc.md) to drive invalidations** — the DB's changelog is the source of invalidation events; if the DB commit succeeded, the invalidation eventually arrives.
- **Switch to write-through** — atomic at the cache layer.
- **Transactional outbox** — write the cache-invalidation message in the same DB transaction as the data change.

### Where cache-aside fits

Cache-aside is the right default when:

- You're using a standalone cache (Redis, Memcached) alongside a DB.
- Reads dominate writes (cache pays off).
- A few seconds of staleness is acceptable.
- You can tolerate occasional cache misses (cold reads are slow but correct).

It's the wrong choice when:

- You need strong read-your-writes consistency — use write-through or skip cache.
- The cache is on the *write* hot path with high QPS — write-through or write-behind may help.
- The cache is a real source of truth (e.g., session store) — that's not a cache, that's a primary store; use a durable backend.

### Real-world scale

Facebook's NSDI 2013 paper, *Scaling Memcache at Facebook*, is the canonical case study. Key takeaways:
- They use cache-aside with Memcached at billions of QPS.
- The hard problems are not GET/SET — they're invalidation, stampedes, regional consistency, and pool partitioning.
- They use **leases** (a special invalidation token) to prevent stale-set races.
- They use **gutter pools** to absorb traffic when a primary cache node fails.

Twitter, Netflix, Pinterest, Discord, and most large web companies use cache-aside with Memcached or Redis as their baseline; the engineering is in the failure modes, not the basic pattern.

### Compared to alternatives

- **[Read-through / write-through / write-behind](cache-managed-strategies.md)**: the cache layer manages reads/writes. Simpler application code, more complex cache infrastructure.
- **Refresh-ahead**: cache proactively refreshes before expiration.
- **CDC-driven invalidation**: the DB's changelog drives cache updates instead of application code.
- **No cache**: sometimes the DB is fast enough; not every read needs caching.

Cache-aside is the most flexible because the application chooses what to cache and when; it's the most error-prone because the application is responsible for getting it right.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Cache-Aside (classic)** | App reads/writes both; invalidate on write. |
| **Cache-Aside + TTL only** | No explicit invalidation; rely on TTL alone (acceptable for very stable data). |
| **Cache-Aside + CDC invalidation** | DB changelog drives invalidations instead of application code. |
| **Versioned cache keys** | Writes increment a version; old entries die naturally. |
| **Two-level cache** | Local in-process cache + shared Redis cache. |
| **Negative caching** | Cache "not found" results too. |
| **[Read-through](cache-managed-strategies.md)** | Cache layer handles MISS-and-load; app sees only the cache. |
| **[Write-through](cache-managed-strategies.md)** | Cache layer writes synchronously to DB. |
| **[Write-behind](cache-managed-strategies.md)** | Cache layer writes async to DB. |
| **Refresh-ahead** | Proactive refresh near expiry. |

## When NOT to use

- **Strong consistency required** for reads — use no cache, or read from the primary.
- **Write-heavy workload** with low cache hit rate — overhead without benefit.
- **Tiny data set** that fits in DB buffer cache anyway — the DB is already caching it.
- **Without TTLs** — eventually you have a stale, unbounded cache and no way to clean it.
- **As a primary store** — caches lose data; if it matters, use a real database.

---

## Real-world implementations

| Tool | Notes |
|---|---|
| **Memcached** | The original distributed cache; simple LRU key-value. |
| **Redis** | Cache + much more; persistence, data structures, pub-sub. |
| **AWS ElastiCache** | Hosted Redis / Memcached. |
| **GCP Memorystore** | Hosted Redis. |
| **Azure Cache for Redis** | Hosted Redis. |
| **DragonflyDB / KeyDB** | Modern Redis-compatible alternatives. |
| **Caffeine** | High-performance in-process Java cache. |
| **Guava Cache** | Older Java in-process cache. |
| **node-cache, cachetools** | Language-specific in-process caches. |
| **Varnish** | HTTP-layer cache; effectively cache-aside for web responses. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Facebook / Meta** | Cache-aside with Memcached at billions of QPS; basis of NSDI 2013 paper. | ✅ Verified — [NSDI 2013 paper](https://research.facebook.com/publications/scaling-memcache-at-facebook/) |
| **Twitter / X** | Multi-tier Memcached/Redis caching for timeline reads. | ✅ Verified — Twitter Engineering blog (multiple posts) |
| **Netflix** | EVCache (Memcached-based) for personalization, viewing history. | ✅ Verified — Netflix Tech Blog, EVCache open-source |
| **Pinterest** | Memcached + Bloom filters for pin/board lookups. | ✅ Verified — Pinterest Engineering blog |
| **Discord** | Cassandra + Redis cache-aside for messages. | ✅ Verified — Discord Engineering blog |
| **Reddit** | Memcached/Redis for listing pages and comment trees. | ✅ Verified — Reddit Engineering posts |
| **GitHub** | Memcached for many read paths. | ⚠ Widely known; specific architecture varies |
| **Almost every web-scale company** | Cache-aside is the default caching pattern. | ✅ Industry standard |

---

## Further reading

- *Scaling Memcache at Facebook* (Nishtala et al., NSDI 2013) — the canonical paper.
- Microsoft Azure Architecture Center — Cache-Aside pattern.
- AWS Database Caching Strategies whitepaper.
- *Designing Data-Intensive Applications* (Kleppmann), Ch on derived data.
- Redis documentation — [Caching strategies](https://redis.io/learn/howtos/solutions/caching-architecture/common-caching).
- *High Performance Browser Networking* (Grigorik) — HTTP caching context.
- Netflix Tech Blog series on EVCache.

---

*Diagram sources: [`../diagrams/src/cache-aside-flow.d2`](../diagrams/src/cache-aside-flow.d2), [`../diagrams/src/cache-aside-pitfalls.d2`](../diagrams/src/cache-aside-pitfalls.d2).*
