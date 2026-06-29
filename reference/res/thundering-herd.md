# Thundering Herd / Cache Stampede

**Aliases:** Cache Stampede, Dog-Pile Effect, Thundering Herd, Hot Key Stampede
**Category:** Resilience / Caching
**Sources:**
[Vattani, Chierichetti, Lowenstein — *Optimal Probabilistic Cache Stampede Prevention* (VLDB 2015)](http://www.vldb.org/pvldb/vol8/p886-vattani.pdf) ·
[Nishtala et al. — *Scaling Memcache at Facebook* (NSDI 2013)](https://research.facebook.com/publications/scaling-memcache-at-facebook/) — the "leases" mechanism ·
[Varnish documentation — Grace mode](https://varnish-cache.org/docs/trunk/users-guide/vcl-grace.html) ·
[Cloudflare blog: stale-while-revalidate](https://blog.cloudflare.com/stale-while-revalidate/) ·
[Wikipedia: Thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)

---

## Problem

> [!TIP]
> **ELI5.** You have a hot cached value (the homepage banner, a JWT public key, a popular product page). Many concurrent requests are reading it from cache. Then the TTL expires (or the cache is flushed, or a node restarts) and the value is suddenly *gone*. Right at that instant, thousands of requests all MISS the cache simultaneously. They all hit the underlying database with the *exact same query*. The database — sized for normal cached load — collapses under the synchronized stampede. Even queries unrelated to the hot key start timing out. **Thundering herd** is the general term; **cache stampede** is its most famous instance.

The setup is universal in cached systems:

- The cache exists *because* the underlying resource (DB, expensive computation, external API) can't handle the full read load.
- The cache is sized to absorb that load when it has the answer.
- The DB is sized for occasional cache misses, not for concurrent misses on hot keys.

When a hot key expires (or is flushed) and many concurrent requests miss simultaneously:

1. T0: 10,000 concurrent requests arrive for the same key.
2. T1: each MISSes the cache.
3. T2: each starts a (potentially identical) DB query.
4. T3: 10,000 concurrent queries hit the DB.
5. T4: DB collapses — connection limits hit, query queue grows, **even unrelated queries time out**.
6. T5: cascading failure — application threads block on DB → app servers exhaust connections → upstream timeouts → retries → more load.

The problem:

![Thundering herd problem flow](../diagrams/svg/thundering-herd-problem.svg)

This isn't a hypothetical edge case. It's one of the most common causes of cache-tier outages in practice:
- TTLs synchronize when entries are set together (deployment, warmup).
- Cache restarts/failovers wipe entries en masse.
- Popular keys have many concurrent readers — that's why they're cached.
- Memcached/Redis evictions can cascade across keys.

The pattern's name — "thundering herd" — comes from earlier OS work where many processes woke simultaneously on a single event (a wake-up storm). Cache stampede is the most famous modern instance.

## How it works (the fix)

> [!TIP]
> **ELI5.** Stop letting 10,000 requests do the same expensive thing at once. Either: (1) **let only the first one do the work** and have the others wait for its result (single-flight), (2) **refresh the value before it expires** so no one ever sees a miss, (3) **return a stale value** while quietly refreshing in the background, or (4) **add randomness** so cache expiries don't synchronize. Most real systems use several of these together.

### The five major mitigations

The mitigations:

![Thundering herd mitigations](../diagrams/svg/thundering-herd-mitigations.svg)

#### 1. Single-flight / request coalescing (the primary fix)

On a cache MISS for key K, the first request acquires a "compute in progress" lock for K. Subsequent concurrent requests for K **wait** for the first to finish, then read its result from the cache. Only one DB query executes regardless of concurrency.

Implementations:
- **Go**: `golang.org/x/sync/singleflight` — the canonical example.
- **Java**: `Caffeine.LoadingCache` (built-in).
- **Python**: `aiocache`, `request_coalescer`, application-level locks.
- **Ruby**: `identity_cache`, `dalli` with `race_condition_ttl`.
- **Distributed coordination**: Redis-based locks (with all the usual distributed-lock caveats).

This pattern alone prevents the vast majority of stampedes — it's the first and most important defense.

```go
import "golang.org/x/sync/singleflight"

var g singleflight.Group

func getUser(id string) (User, error) {
    v, err, _ := g.Do(id, func() (any, error) {
        return loadUserFromDB(id)
    })
    return v.(User), err
}
// If 10,000 goroutines call getUser("alice") concurrently:
// - First one executes loadUserFromDB
// - Other 9,999 wait for first; share its result
// - One DB query, not 10,000
```

#### 2. Probabilistic early expiration (XFetch / Beta algorithm)

Each read of a value near its TTL has a small (and growing) probability of being chosen to refresh **early**, before the actual expiry. This distributes refreshes across time, so there's no synchronized expiry moment.

The algorithm (Vattani et al., VLDB 2015 — *Optimal Probabilistic Cache Stampede Prevention*):

```
on read of (value, expiry, delta):
    if now - (delta * beta * ln(random())) > expiry:
        refresh in background
    return value
```

`delta` is the expected recomputation time; `beta` is a tunable factor (typically 1.0). As `now` approaches `expiry`, the probability of an early refresh rises.

No locking required. Statistically smooth distribution of refreshes. Provably near-optimal for cost.

#### 3. Background refresh (refresh-ahead)

Identify hot keys; refresh them on a schedule **before** they expire. Application reads always hit a populated cache; background worker handles all DB load on a predictable schedule.

Works well for:
- Known hot keys (homepage, popular products, configuration).
- Stable workloads.

Doesn't help with:
- Long-tail or unpredictable hot keys.
- Sudden traffic spikes on previously-cold keys.

Used by Caffeine's `refreshAfterWrite`, custom background-refresh workers in many production systems.

#### 4. Serve-stale-while-refresh

Keep an **old** cached value past its expiry. On a MISS-after-expiry:
- First request triggers an **async** refresh.
- All requests (including the trigger) get the **stale** value immediately.
- Once refresh completes, future reads get fresh data.

This trades a bounded staleness window for completely eliminating the latency spike of a cold lookup. It's the basis of:

- HTTP `Cache-Control: stale-while-revalidate` (RFC 5861) — used by browsers, CDNs (Cloudflare, Vercel), and shared caches.
- Varnish's **grace mode**: serve stale up to a grace period.
- Nginx's `proxy_cache_use_stale`.
- Application-level "serve stale on backend error" logic.

For most read-heavy systems, this is acceptable — users see a value that's slightly stale (seconds or minutes) instead of waiting for a fresh fetch.

#### 5. TTL jitter (add entropy)

Don't use TTL = exactly 300s for everything. Use TTL = 300 + random(0, 30)s. Even if many entries are written at the same time (deployment, warmup, mass eviction), their expiries are spread over a window — no synchronized expiry storm.

```python
# Bad: synchronized expiry
cache.set(key, value, ttl=300)

# Good: jittered expiry
cache.set(key, value, ttl=300 + random.randint(0, 30))
```

Easy. Cheap. Prevents an entire class of stampedes. Always do this.

### Layering the defenses

Production systems usually combine several:

- **TTL jitter** everywhere (zero cost).
- **Single-flight** for hot operations (mostly transparent via libraries).
- **Stale-while-revalidate** for read-heavy endpoints (HTTP caches, application caches).
- **Probabilistic early expiration** or **background refresh** for known hot keys.

### Facebook's leases (a sophisticated variant)

Facebook's NSDI 2013 paper introduced **leases** in Memcache: when a client MISSes a key, it receives a lease (a token). Only the client with the lease may write to that key. Other clients trying to write are told "another client has the lease — wait or use stale." This is essentially single-flight implemented at the cache layer, plus a serve-stale fallback. It's the most thorough solution from a paper that's worth reading for the broader cache-engineering wisdom.

### Thundering herd in non-cache contexts

The pattern generalizes:

- **OS wakeups**: many processes wake on a single event; modern kernels solve this with `EPOLLEXCLUSIVE`, `accept_mutex`, etc.
- **Kafka consumer rebalances**: many consumers all rejoining and reading offsets at once.
- **Retry storms**: many clients all retrying at the same backoff time → use **jittered exponential backoff**.
- **DNS lookups**: many requests for the same uncached name → single-flight or local DNS cache.
- **Lock contention**: many threads waiting on the same mutex → use sharded locks or read-write locks.
- **Connection storms after a deploy**: many clients reconnecting simultaneously → use jittered reconnect delays.

All of these have the same shape: a synchronized event triggers a burst of identical work that overwhelms a shared resource. The same family of fixes applies: jitter, coalescing, backoff.

### Recognizing it in production

Signs of cache stampede:

- DB load spikes that correlate with cache TTLs or cache restarts.
- Tail latency (p99, p99.9) shoots up while p50 is fine — most requests served from cache; a few experiencing the stampede.
- Cascading timeouts: app metrics show DB timeouts → app thread exhaustion → upstream timeouts.
- After cache failover or restart, "warming" period of high DB load.

A well-instrumented cache layer should expose hit/miss rates, MISS-by-key distribution (top-K), and concurrency on MISS — observability matters for catching these patterns.

### Pre-warming

For cold caches (after deploy, after restart), avoid throwing live traffic at a cold cache. Either:

- **Pre-warm** the cache with synthetic loads before serving real traffic.
- **Gradual ramp**: route a small percentage of traffic first; let the cache populate; ramp up.
- **Replicate cache state** from a peer (Memcached can serialize state; Redis has snapshots).

Otherwise, every cold cache event is potentially a thundering herd.

### Costs and trade-offs

The mitigations have costs:

- **Single-flight** adds latency for waiters (they wait for one upstream call), and requires coordination state. Distributed single-flight (across many app instances) is harder than single-process.
- **Background refresh** wastes compute on values nobody's reading.
- **Stale-while-revalidate** serves out-of-date values; not acceptable for all use cases.
- **Probabilistic refresh** is harder to reason about; "did this refresh fire when I expected?" can be confusing.
- **TTL jitter** is cheap but produces wider expiry spread.

The combination usually wins: jitter (free, always do it) + single-flight (cheap, broad coverage) + stale-while-revalidate or probabilistic refresh for high-traffic paths.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Single-flight / coalescing** | Only first request executes; others wait. |
| **Facebook leases** | Cache-level coordination with lease tokens. |
| **Probabilistic early expiration (XFetch)** | Random early refresh; no locks. |
| **Background refresh / refresh-ahead** | Scheduled pre-expiry refresh. |
| **Serve-stale-while-revalidate** | Bounded staleness in exchange for no MISSes. |
| **TTL jitter** | Random spread of expiries. |
| **Pre-warming / gradual ramp** | Don't expose cold caches to full traffic. |
| **Negative caching** | Cache "not found" to prevent enumeration-stampede. |
| **Bulkhead** ([page](bulkhead.md)) | Limits concurrency per dependency; bounds stampede impact. |
| **Circuit breaker** ([page](circuit-breaker.md)) | Bails out when DB is in trouble. |

## When NOT to use the mitigations

- **Low traffic / cold cache acceptable** — overhead exceeds benefit.
- **Strict freshness requirements** — stale-while-revalidate not acceptable.
- **Tiny key set entirely in cache** — no MISSes possible.
- **When the DB can handle full load** — caching may not be needed at all.

---

## Real-world implementations

| Tool / Pattern | Notes |
|---|---|
| **golang.org/x/sync/singleflight** | Go standard library extension. |
| **Caffeine (Java)** | Built-in single-flight via `LoadingCache`. |
| **Guava Cache (Java)** | Single-flight loader. |
| **dalli (Ruby)** | `race_condition_ttl` for stampede prevention. |
| **identity_cache (Shopify, Ruby)** | Coalescing. |
| **aiocache (Python)** | Async single-flight. |
| **request-coalescer (Node.js)** | npm packages. |
| **Varnish** | Grace mode + serve-stale. |
| **Nginx** | `proxy_cache_use_stale` + `proxy_cache_lock`. |
| **Cloudflare / Vercel CDN** | Native `stale-while-revalidate`. |
| **Facebook Mcrouter** | Implements lease mechanism. |
| **Hystrix** | Bulkhead + circuit breaker; supports request coalescing. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Facebook / Meta** | Leases in Memcache; NSDI 2013 paper. | ✅ Verified — [NSDI 2013](https://research.facebook.com/publications/scaling-memcache-at-facebook/) |
| **Cloudflare** | Stale-while-revalidate at edge; published blog posts. | ✅ Verified — [Cloudflare blog](https://blog.cloudflare.com/stale-while-revalidate/) |
| **Vercel** | Built-in stale-while-revalidate. | ✅ Verified — Vercel docs |
| **Shopify** | identity_cache library; addresses stampedes. | ✅ Verified — Shopify open-source |
| **Discord** | Cassandra + Redis with single-flight patterns. | ✅ Verified — Discord engineering blog |
| **GitHub** | Memcached with race-condition mitigation. | ⚠ Common pattern; specific posts vary |
| **Twitter / X** | Single-flight at scale for popular timeline reads. | ⚠ Mentioned in talks |
| **Almost every large web service** | Some combination of these mitigations. | ✅ Industry standard |

---

## Further reading

- Vattani, Chierichetti, Lowenstein, *Optimal Probabilistic Cache Stampede Prevention* (VLDB 2015) — the XFetch algorithm.
- Nishtala et al., *Scaling Memcache at Facebook* (NSDI 2013) — leases and other production techniques.
- HTTP RFC 5861 — `stale-while-revalidate` and `stale-if-error`.
- Cloudflare blog: stale-while-revalidate.
- Varnish documentation: grace mode.
- Nginx caching configuration documentation.
- Go's `singleflight` package source — short, clear reference implementation.

---

*Diagram sources: [`../diagrams/src/thundering-herd-problem.d2`](../diagrams/src/thundering-herd-problem.d2), [`../diagrams/src/thundering-herd-mitigations.d2`](../diagrams/src/thundering-herd-mitigations.d2).*
