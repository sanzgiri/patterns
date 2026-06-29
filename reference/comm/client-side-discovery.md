# Client-Side Service Discovery

**Aliases:** Client-Side LB, Smart Client Discovery, In-Process Discovery
**Category:** Service discovery
**Sources:**
[Chris Richardson — microservices.io: Client-Side Discovery](https://microservices.io/patterns/client-side-discovery.html) ·
[Microsoft Azure Architecture Center — Service Discovery](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/interservice-communication) ·
Discussion of Netflix Ribbon, Eureka, and Spring Cloud LoadBalancer

---

## Problem

> [!TIP]
> **ELI5.** A caller needs to send a request to one of N healthy instances of `orders-svc`. The naive way: put a load balancer in front of all the instances; the caller just calls the LB. That works, but it costs an extra network hop and gives up control over which instance the caller talks to. The **client-side discovery** alternative: the caller queries the registry directly to get the list of healthy instances and picks one itself. No extra hop; smart caller-side decisions possible (prefer same-zone, prefer fastest, etc.).

When a service caller needs to reach an instance of a downstream service, the basic approach uses a proxy/load balancer ([server-side discovery](server-side-discovery.md)): the caller hits the LB, the LB picks an instance and forwards. This is universal and language-agnostic but has costs:

- **Extra network hop** — every call goes through the LB, adding latency and a bandwidth concentration point.
- **LB is critical infrastructure** — its outage takes down all traffic; it must be highly available.
- **Limited intelligence** — the LB doesn't know the caller's context (its zone, its current load, its preferences). Routing decisions are based on what the LB sees, not what the caller knows.

**Client-side discovery** moves the responsibility to the caller itself: the caller queries the registry, receives the list of healthy instances, and makes its own routing decision. The caller's code (or sidecar library) does the load balancing.

## How it works

> [!TIP]
> **ELI5.** The caller has a "discovery client" library. On startup (and continuously), it asks the registry for the list of healthy `orders-svc` instances. When the app wants to call `orders-svc`, the library picks one of those instances using a load-balancing strategy (round-robin, random, prefer-same-zone) and connects directly. No proxy in the middle. The library watches the registry for changes so the list stays fresh.

The flow:

![Client-side service discovery](../diagrams/svg/client-side-discovery.svg)

The caller's discovery client (a library or sidecar):

1. **Queries the registry** for healthy instances of the target service. Caches the result locally (typically 10–60 seconds).
2. **Watches** the registry for changes (using long-polling, gRPC streams, or similar) so the cache stays fresh without polling.
3. **On each call**: applies a load-balancing strategy to the cached list and connects directly to the chosen instance.
4. **Tracks per-instance health** locally — if direct calls reveal an instance is bad (returning errors, slow), the client can temporarily evict it even before the registry notices.

The connection from caller to chosen instance is **direct** — no intermediate proxy. The registry is consulted to find addresses; once known, the caller connects without further indirection.

### Load-balancing strategies

The caller can use a wide variety of strategies, since it knows context the central LB doesn't:

- **Round-robin** — simplest, fairly distributes.
- **Random** — minimal state needed; statistically fair.
- **Least-connections** — picks instance with fewest in-flight requests from this caller.
- **Latency-aware** — keeps recent latency stats per instance; prefers faster ones.
- **Zone-aware** — prefers instances in the caller's own availability zone (saves cross-AZ latency and cost).
- **Power-of-two-choices** — randomly pick 2, send to the less-loaded. Provably good worst case.
- **Weighted** — instance has weight (e.g., based on capacity); picks proportionally.
- **Consistent hashing** — same input always routes to same instance (for cache locality).

The richness of strategies is one of client-side discovery's main strengths. A central LB typically supports a small fixed set; a client library can experiment per-service with the best strategy.

### Trade-offs

The advantages:

- **No extra network hop.** Caller talks directly to the chosen instance.
- **Smarter routing.** Caller knows its own zone, current load, preferences; central LB doesn't.
- **No LB infrastructure to operate.** Saves operational complexity and SPOF risk.
- **Fine-grained control per call.** Retry to a different instance on failure; prefer instances with the same trace, etc.

The disadvantages:

- **Every caller needs a discovery client library** — in every language. Polyglot orgs maintain N implementations with drift risk.
- **Cache freshness is the caller's problem.** Stale caches lead to routing to dead instances briefly. Watch APIs mitigate but don't eliminate.
- **LB logic duplicated** across all callers. A bug or improvement must propagate to every service.
- **Couples callers to the registry's protocol.** If you migrate from Eureka to Consul, every service's library changes.
- **Authentication and policy are caller-side too** — harder to enforce centrally.

These costs are why **service mesh** has emerged as the dominant evolution: a per-pod sidecar (Envoy) does client-side LB on behalf of the local app, while the app itself sees only a stable address. The app gets server-side simplicity; the sidecar gets client-side sophistication.

### Implementation choices

- **As a library in the application** — Spring Cloud LoadBalancer, gRPC's name resolution + LB plugin, Finagle (Twitter), Hystrix + Ribbon (Netflix OSS). Tight integration with the app.
- **As a sidecar process** — Envoy doing client-side LB on behalf of the app. Language-agnostic but adds the sidecar hop (small).
- **As a daemon on every node** — kube-proxy historically did this (now mostly replaced by IPVS or eBPF in modern K8s clusters).

### Caching freshness vs staleness

The fundamental tension: how often should the client refresh its instance list? Too often, you spam the registry. Too rarely, you call dead instances.

Common pattern:

- **Initial fetch** at startup; cache for 10–60 seconds.
- **Watch the registry** for change events; update cache reactively.
- **On call failure** (connection refused, 5xx burst), force a refresh.
- **Track per-instance success/failure rates** locally; evict bad instances temporarily even if registry says they're healthy.

This blended approach is what mature client libraries (Spring Cloud, gRPC, Envoy) actually do.

### When client-side is the right choice

Client-side discovery is the right call when:

- Your org is mostly one language (so one good library covers most callers).
- Latency is critical and you can't afford the LB hop.
- You want smart, context-aware routing decisions.
- You have the engineering capacity to maintain and evolve the client library.

It's the wrong call when:

- You're polyglot (Python, Go, Java, Node, Rust) — the maintenance cost of N libraries dominates.
- Centralized policy and observability are essential.
- Latency budget can absorb one extra hop (most apps).
- You're already running a service mesh (which gives you the best of both worlds).

In modern practice, **pure client-side discovery is becoming rare** — most architectures have shifted to either server-side via cloud LBs or mesh-based via Envoy sidecars. Client-side discovery as an application-level concern is now mostly seen in Spring Cloud apps and the Netflix-OSS legacy ecosystem.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **In-process library** | Discovery + LB in app's address space (Ribbon, Finagle, gRPC LB). |
| **Sidecar-based client LB** | Envoy/Linkerd does client-side LB transparently. |
| **DNS-only client LB** | Caller uses DNS, picks among A-records itself. Limited intelligence but universal. |
| **Round-robin** | Simplest LB strategy. |
| **Latency-aware** | Tracks recent per-instance latency. |
| **Zone-aware** | Prefers same-zone instances. |
| **Power-of-two-choices** | Randomly pick 2, send to less-loaded. |
| **Consistent hashing** | Same input → same instance (cache locality). |
| **Server-side discovery** | The alternative; LB in front does the routing. |
| **Service mesh** | Hybrid that combines both views. |

## When NOT to use

- **Polyglot orgs without resources for multi-language clients** — server-side or mesh is cheaper.
- **When you need centralized policy** that all callers must obey.
- **When the network hop is acceptable** — most apps tolerate it.
- **Without engineering capacity to maintain the client library** at production quality.

---

## Real-world implementations

| Implementation | Notes |
|---|---|
| **Spring Cloud LoadBalancer** | Java; the successor to Ribbon; common in Spring Boot apps. |
| **Netflix Ribbon (deprecated)** | The historical reference; superseded by Spring Cloud LoadBalancer. |
| **gRPC client LB** | Native gRPC support for client-side LB plugins (round-robin, pick-first, custom). |
| **Finagle** | Twitter's open-source RPC stack with client-side LB. |
| **Eureka client** | Combined with Ribbon historically; still in some Spring Cloud apps. |
| **Consul client (DNS or HTTP)** | Either DNS-based or via Consul client library. |
| **Envoy sidecar (mesh)** | Does client-side LB on behalf of app. |
| **HAProxy embedded** | Some apps embed HAProxy as a library/sidecar for client-side LB. |
| **HashiCorp's Sockaddr** | Helper library for client-side address resolution. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Netflix (historical)** | Eureka + Ribbon was the canonical client-side discovery stack; now mostly K8s-native. | ✅ Verified — Netflix OSS history; Netflix Tech Blog |
| **Twitter** | Finagle + ZooKeeper for client-side LB; long-standing. | ✅ Verified — Finagle open source, Twitter Engineering posts |
| **Lyft (early Envoy)** | Original Envoy was a client-side LB sidecar; evolved into mesh. | ✅ Verified — Lyft Engineering blog |
| **Many Spring Cloud users (banks, retailers, telcos)** | Spring Cloud LoadBalancer in widespread use. | ✅ Verified — Spring docs and case studies |
| **gRPC ecosystem (Google internal, gRPC users)** | gRPC's built-in client LB is widely used. | ✅ Verified — gRPC docs |

---

## Further reading

- Chris Richardson, *microservices.io* — Client-Side Discovery pattern entry.
- Spring Cloud LoadBalancer documentation — concrete Java implementation.
- gRPC's name resolution and load balancing documentation.
- Netflix Tech Blog — historical Ribbon/Eureka posts.
- Buoyant's Linkerd blog — modern mesh-based take on client-side LB.
- Newman, *Building Microservices* (2nd ed.) — comparison of discovery options.
- Adrian Cockcroft's QCon talks — pre-mesh microservices wisdom.

---

*Diagram sources: [`../diagrams/src/client-side-discovery.d2`](../diagrams/src/client-side-discovery.d2), [`../diagrams/src/discovery-comparison.d2`](../diagrams/src/discovery-comparison.d2).*
