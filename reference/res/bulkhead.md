# Bulkhead

**Aliases:** Resource Isolation, Compartmentalization, Bounded Pool, Failure Domain
**Category:** Resilience
**Sources:**
[Microsoft Azure — Bulkhead pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead) ·
Michael T. Nygard, *Release It!* (2nd ed., 2018), Ch on Stability Patterns ·
[Netflix Hystrix wiki — Bulkhead](https://github.com/Netflix/Hystrix/wiki/How-it-Works#how-and-when-isolation-strategies-are-used) ·
[Resilience4j documentation](https://resilience4j.readme.io/docs/bulkhead) ·
[AWS Builders' Library — Workload isolation using shuffle-sharding](https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/)

---

## Problem

> [!TIP]
> **ELI5.** The name comes from ship design — ships are built with watertight **bulkheads** between compartments, so if one compartment floods, the others stay dry and the ship doesn't sink. Apply this to software: when one downstream dependency starts misbehaving (slow responses, errors), it can drain shared resources (threads, connections, memory) and bring down requests that have nothing to do with the broken dependency. The fix: isolate resources so failure in one place cannot exhaust the resources needed for other work.

The classic failure mode this pattern prevents:

1. Your app server has a 200-thread HTTP request pool.
2. Service B (one of many downstreams) starts responding slowly — 5 seconds per call instead of 100ms.
3. Requests to B occupy threads for 50× longer than usual.
4. Soon, *all 200 threads* are blocked waiting on B.
5. Requests for Service A and Service C (which are healthy!) can't get a thread.
6. Your entire app is down because B is slow.

This is **cascading failure via resource exhaustion**. One bad neighbor takes down the building.

The same dynamic plays out at many levels:

- **Database connections**: one slow query holds a connection; queue grows; all queries time out.
- **Thread pools**: one slow downstream holds threads; other paths starve.
- **Memory**: one runaway workload exhausts heap; everything fails.
- **CPU**: one busy-loop saturates cores; everything is slow.
- **Network**: one heavy talker saturates bandwidth; others time out.
- **Tenants in a multi-tenant system**: one noisy customer's workload starves the rest.
- **Files / file descriptors**: one leaky workload exhausts FDs; new connections fail.

The **bulkhead pattern** is the general answer: separate resources by **isolation domain** so one domain's failure cannot drain resources needed by others.

The naming is from Michael Nygard's *Release It!* (2007), which codified the pattern as a foundational stability tactic — though the underlying idea (partition resources to bound failure) is much older in operating systems and distributed computing.

## How it works

> [!TIP]
> **ELI5.** Don't give all your downstream calls one shared pool of resources. Give each downstream (or each tenant, or each workload class) its own pool, sized to its expected load. When one pool fills up because its downstream is slow, only calls to that downstream are affected — others still have full resources.

The basic concept:

![Bulkhead: with vs without isolation](../diagrams/svg/bulkhead.svg)

A canonical implementation:

```java
// Without bulkhead: one shared HTTP client / thread pool
HttpClient sharedClient = HttpClient.newBuilder()
    .executor(Executors.newFixedThreadPool(200))
    .build();

// With bulkhead: per-dependency pools
HttpClient clientForServiceA = HttpClient.newBuilder()
    .executor(Executors.newFixedThreadPool(50))   // dedicated 50 threads
    .build();
HttpClient clientForServiceB = HttpClient.newBuilder()
    .executor(Executors.newFixedThreadPool(50))
    .build();
HttpClient clientForServiceC = HttpClient.newBuilder()
    .executor(Executors.newFixedThreadPool(50))
    .build();
```

Now if Service B's calls hang, they fill up only `clientForServiceB`'s pool. Calls to A and C continue normally.

### Bulkhead at multiple levels

The pattern applies recursively at many scales:

![Bulkhead isolation at six levels](../diagrams/svg/bulkhead-levels.svg)

**1. Thread-pool bulkhead** (within one process): separate thread pool per dependency.
- Library support: Resilience4j Bulkhead, Hystrix (legacy), Polly (.NET), Failsafe.
- Cost: per-pool memory overhead; managing many pools.
- Benefit: classic, most common form.

**2. Semaphore bulkhead** (same thread, count limit): bound concurrency without dedicated threads.
- Async/reactive code where threads are virtual (Project Loom, Go goroutines, Node.js).
- Lighter than thread pools (no context switch).
- Doesn't help if the thread itself is blocked.

**3. Process bulkhead** (separate processes per role): split risky workloads into separate processes.
- Misbehaving process OOMs or crashes without taking out neighbors.
- Common when refactoring legacy code; security isolation.
- Browser tab isolation (Chrome) is a famous example.

**4. Tenant / workload bulkhead**: separate compute/resources per tenant or workload class.
- Noisy-neighbor tenant can't exhaust shared resources.
- Kafka per-topic / per-user quotas; database per-user connection limits; per-tenant request pools.
- Foundational for multi-tenant SaaS.

**5. Cluster / deployment bulkhead**: separate clusters or deployment "cells" per workload class.
- Production segregated from staging; critical from non-critical.
- Microsoft's "deployment stamps" pattern; AWS's cell-based architecture; Slack's "cells".
- Slack has publicly described its cellular architecture for blast-radius control.

**6. Availability-zone / region bulkhead**: distinct AZ/regions with no shared infrastructure.
- AWS S3 outage in us-east-1 doesn't affect us-west-2.
- The standard high-availability recommendation from every cloud provider.
- Required for serious uptime commitments.

These compound: a serious system has bulkheads at multiple levels (per-dependency thread pools *within* per-tenant deployments *within* multi-region cells).

### Sizing the bulkheads

The hard practical question: **how big should each bulkhead be?** Too small → underutilization; too big → poor isolation.

Heuristics:
- **Little's Law**: average concurrent requests = arrival rate × average response time. Size pools to cover the expected concurrency at p99 latency.
- **Per-dependency observation**: measure each downstream's QPS and latency; size to handle 2-3× peak load.
- **Capacity planning by tier**: critical services get bigger pools than nice-to-haves.
- **Headroom for degradation**: pool should handle peak + some slack for slow-but-not-failing periods.

In practice, start conservative (small pool), increase when you observe saturation that *isn't* due to a failing downstream. Hystrix's defaults were quite small (10-thread pools) precisely to force you to make this trade-off explicit.

### Bulkhead + circuit breaker

[Circuit Breaker](circuit-breaker.md) and Bulkhead are complementary:

- **Bulkhead** prevents one slow dependency from exhausting shared resources.
- **Circuit Breaker** detects that the dependency is failing and stops sending requests to it entirely.

Together: a failing dependency's bulkhead fills up (per-dependency pool exhausted), the circuit breaker detects this (request rejections / latencies), and trips — now requests to that dependency fail fast instead of queuing. Other dependencies continue uninterrupted.

These two patterns together cover most of the "one bad dependency takes down everything" scenarios.

### Bulkhead + load shedding + backpressure

When a bulkhead's pool is full, new requests must be:
- **Rejected** (load shedding — return 503 immediately).
- **Queued** (bounded queue — risks adding latency but smooths bursts).
- **Backpressured** (propagate the slow signal upstream — see [Backpressure](backpressure.md)).

The choice depends on what's calling you:
- **Stateless HTTP API**: usually reject (load shed); caller can retry.
- **Async pipeline / queue consumer**: usually queue with bound; let upstream queue grow within limits.
- **Real-time stream**: usually drop oldest or backpressure upstream.

### Shuffle sharding: a powerful variant

AWS popularized **shuffle sharding** in their Builders' Library: instead of dedicating one shard per tenant (expensive) or fully sharing (no isolation), give each tenant a *random subset* of shards. With N shards and each tenant assigned to K of them:

- Two random tenants share all K shards only with probability `(K/N)^K` — usually tiny.
- A misbehaving tenant degrades only the K shards it's assigned to.
- Probability a *given* other tenant is fully co-located is tiny.

This is the foundation of Route53's "horizontal" isolation. With 8 shards and K=2, you have C(8,2) = 28 possible tenant assignments. The probability two random tenants share all shards is 1/28 ≈ 3.5%. With more shards and K, it drops sharply.

Shuffle sharding gives much of the benefit of per-tenant isolation at much lower cost.

### Trade-offs

Advantages:
- **Bounded blast radius**: failure stays in one bulkhead.
- **Predictable degradation**: a bulkhead full → known requests fail/queue.
- **Composes with other resilience patterns**: circuit breaker, load shedding, etc.
- **Works at any scale**: from thread pools to whole regions.

Disadvantages:
- **Resource overhead**: per-bulkhead pools, queues, monitoring.
- **Sizing complexity**: how much for each? Wrong answer → underutilization or no isolation.
- **Operational overhead**: more pools to monitor; more knobs to tune.
- **Can mask real capacity problems**: if pool is always full, is the dependency slow or are you under-provisioned?

### Anti-patterns

- **One shared pool for everything**: defeats the pattern.
- **Pools sized for happy path only**: any slow downstream fills them instantly.
- **No monitoring on bulkhead utilization**: can't detect when a pool is saturated.
- **No timeouts on calls within a bulkhead**: slow calls hold pool capacity forever.
- **Hidden shared resources**: per-dependency thread pools but one shared connection pool to the DB → DB connections become the new shared bottleneck.

### Compared to alternatives

- **Just adding capacity**: bigger pools help temporarily but don't isolate; one bad dependency still saturates them.
- **Async / non-blocking I/O alone**: removes the thread bottleneck but not the connection-pool or memory bottleneck.
- **Per-request timeouts only**: bounds individual request latency but lets aggregate load still drain resources.
- **Backpressure only**: useful but doesn't isolate downstream failures from each other.

Bulkhead works best when *combined* with these — it isolates, others bound or signal.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Thread-pool bulkhead** | Separate thread pools per dependency. |
| **Semaphore bulkhead** | Concurrency count limit; no separate threads. |
| **Connection-pool bulkhead** | Separate connection pools per DB/dependency. |
| **Process bulkhead** | Separate processes for isolation. |
| **Tenant bulkhead** | Per-tenant resource isolation. |
| **Cell-based architecture** | Bulkhead at the deployment level (Slack, AWS cells). |
| **Shuffle sharding** | Random subset of shards per tenant for cheap-ish isolation. |
| **[Circuit breaker](circuit-breaker.md)** | Complementary; detects failure and stops sending. |
| **[Rate limiting / throttling](throttling.md)** | Bounds inbound; bulkhead bounds resource use. |
| **[Backpressure](backpressure.md)** | Propagates load signal upstream. |

## When NOT to use

- **Trivial workload** with one or two dependencies — overhead exceeds benefit.
- **No resource contention possible** — single-threaded, single-dependency code.
- **When you can't measure** to size the bulkheads — wrong sizes are worse than no bulkhead.
- **In place of fixing the actual slow dependency** — bulkhead masks the underlying problem; still address the root cause.

---

## Real-world implementations

| Tool | Notes |
|---|---|
| **Resilience4j Bulkhead** | Java; ThreadPoolBulkhead and SemaphoreBulkhead. |
| **Hystrix (legacy)** | Netflix's original; now in maintenance mode. |
| **Polly (.NET)** | Bulkhead policy. |
| **Failsafe (Java)** | Includes bulkhead. |
| **Sentinel (Alibaba)** | Bulkhead-style resource isolation. |
| **gobreaker, sony/gobreaker** | Go circuit-breaker (often combined with semaphore isolation). |
| **AWS SDK adaptive retry** | Built-in backpressure / bulkhead behavior. |
| **Envoy / Istio** | Connection limits, max requests per upstream — implicit bulkhead. |
| **Kubernetes resource limits** | Per-container CPU/memory caps act as bulkhead. |
| **Linux cgroups** | OS-level resource isolation. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Netflix** | Hystrix pioneered the thread-pool bulkhead pattern in microservices. | ✅ Verified — Netflix Tech Blog, Hystrix |
| **AWS** | Cell-based architecture for Route 53, S3, DynamoDB; shuffle sharding. | ✅ Verified — [AWS Builders' Library](https://aws.amazon.com/builders-library/) |
| **Slack** | Cellular architecture for blast-radius control. | ✅ Verified — Slack Engineering blog |
| **Microsoft Azure** | Deployment stamps; per-region cells. | ✅ Verified — Azure Architecture Center |
| **Alibaba** | Sentinel for resource isolation across services. | ✅ Verified — Sentinel docs / Alibaba blog |
| **Google** | Per-job, per-user resource isolation in Borg / SRE practices. | ✅ Verified — SRE book |
| **Lyft, Uber, Airbnb** | Envoy mesh with per-upstream limits. | ✅ Verified — Lyft Envoy origin, Uber/Airbnb engineering blogs |
| **Stripe** | Per-merchant rate limits and resource isolation. | ✅ Verified — Stripe engineering blog |
| **GitHub** | Per-customer rate limits + tenant isolation. | ✅ Verified — GitHub status / engineering posts |
| **Most major SaaS platforms** | Multi-tenant isolation is universal. | ✅ Industry standard |

---

## Further reading

- Michael Nygard, *Release It!* (2nd ed.) — the foundational text for stability patterns.
- Microsoft Azure — Bulkhead pattern article.
- AWS Builders' Library — *Workload isolation using shuffle-sharding* — practical detail.
- *Site Reliability Engineering* (Google SRE book) — Ch on overload, capacity planning.
- *Building Microservices* (Newman, 2nd ed.) — discusses bulkheads in microservice context.
- Resilience4j documentation.
- Netflix Tech Blog (Hystrix archive).
- AWS re:Invent talks on cell-based architecture.

---

*Diagram sources: [`../diagrams/src/bulkhead.d2`](../diagrams/src/bulkhead.d2), [`../diagrams/src/bulkhead-levels.d2`](../diagrams/src/bulkhead-levels.d2).*
