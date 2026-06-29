# Server-Side Service Discovery

**Aliases:** Server-Side LB, Proxy-Based Discovery, Discovery via Load Balancer
**Category:** Service discovery
**Sources:**
[Chris Richardson — microservices.io: Server-Side Discovery](https://microservices.io/patterns/server-side-discovery.html) ·
[Microsoft Azure Architecture Center — Service Discovery](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/interservice-communication) ·
Kubernetes documentation — Service and kube-proxy

---

## Problem

> [!TIP]
> **ELI5.** With [client-side discovery](client-side-discovery.md), every caller needs a library that knows how to find services. In a polyglot org, that's N libraries to write and maintain. The alternative: put a load balancer in front of each service. Callers just send their request to a stable address (DNS name or virtual IP); the LB figures out which healthy instance to forward to. Callers stay dumb — they don't even know discovery is happening.

In a polyglot microservices architecture, each language ecosystem typically has different conventions, libraries, and limitations. Building a service-discovery library for each of them is expensive:

- Python clients need an `asyncio`-aware version that integrates with their HTTP libraries.
- Java needs Spring or gRPC integrations.
- Go needs to fit `net/http` and gRPC patterns.
- Node, Ruby, .NET, Rust — each needs its own.

And each of those libraries needs:

- Registry-protocol support (Eureka, Consul, etcd).
- Load-balancing strategies (round-robin, zone-aware, latency-aware).
- Caching with TTL and watch-based invalidation.
- Health-tracking and circuit-breaking.
- Per-call retry-to-alternate logic.

Maintaining feature parity across N languages is a constant tax. And every service-team developer must learn the library, configure it, and debug it when things go wrong.

**Server-side discovery** moves all of this into infrastructure. A load balancer or proxy sits in front of each service. Callers send requests to a fixed address — typically a DNS name or virtual IP that resolves to the LB. The LB queries the registry, picks an instance, and forwards. Callers don't need a discovery library at all; they just call the name.

## How it works

> [!TIP]
> **ELI5.** Set up a load balancer with the name `orders-svc`. The LB knows about all the healthy instances of orders-svc (kept in sync from the registry). Callers just `http://orders-svc/...` and the LB forwards to one of the live instances. Caller code is plain HTTP/gRPC — no discovery library, no awareness that there's a registry behind the scenes. The LB takes the extra network hop, but you trade that for radical simplicity in caller code.

The flow:

![Server-side service discovery](../diagrams/svg/server-side-discovery.svg)

The caller treats the downstream service as a single address (DNS name, virtual IP, or hostname behind a load balancer). The load balancer:

1. **Maintains the list** of healthy instances — typically by querying or being pushed updates from the service registry.
2. **Receives the caller's request** at the fixed address.
3. **Picks an instance** using its configured load-balancing strategy.
4. **Forwards** the request and returns the response.

The caller has no idea any of this is happening. From its perspective, `orders-svc` is just a hostname.

### Implementations

The "load balancer" can be many things:

- **Cloud-managed LB**: AWS ALB/NLB, Azure Load Balancer, GCP Internal Load Balancer. Cloud platform's responsibility; integrates with registries (e.g., target groups auto-updated from ECS or auto-scaling groups).
- **Kubernetes Service**: K8s' built-in Service abstraction. `kube-proxy` (or IPVS or eBPF in newer K8s) on every node maintains routing rules; the caller hits a virtual IP and the kernel forwards. The implementation is registry-aware and updates as pods come/go.
- **NGINX / HAProxy with dynamic config**: an LB that pulls config from Consul or a similar registry and reloads as services change.
- **Envoy with control plane**: Envoy as a proxy that gets endpoint updates from a control plane (essentially the service-mesh model when used at the inbound edge of each service).
- **API Gateway** (Kong, Apigee, AWS API Gateway): for north-south traffic, the gateway itself does server-side discovery to route to internal services.

In modern K8s deployments, the **Kubernetes Service** is the dominant implementation. Every service has a stable `<name>.<namespace>.svc.cluster.local` DNS name; callers use that name; the kernel routing handles the rest. Effectively zero work in the caller.

### Trade-offs

The advantages:

- **No discovery code in callers** — works for any language equally. Polyglot orgs love this.
- **LB logic centralized** — one place to upgrade, debug, tune.
- **Operationally simpler** — no per-service library to maintain.
- **Caller's network stack is plain HTTP/gRPC** — easy to test, easy to debug.
- **Easy integration with security** — the LB can enforce auth, mTLS, rate limits centrally.

The disadvantages:

- **Extra network hop** — every call goes through the LB. Adds latency (~0.1–1 ms typically) and bandwidth concentration.
- **LB is critical infrastructure** — its outage takes down all traffic. Must be highly available.
- **Less intelligent routing** — the LB doesn't know the caller's zone, current load, preferences. Routing decisions are based on what the LB sees, not what the caller knows.
- **Throughput limited by LB capacity** — at huge scale, the LB becomes a bottleneck. Most cloud LBs scale very well, but it's a consideration.

These trade-offs are typically acceptable for the vast majority of microservices. The simplicity gain — zero discovery code in N languages — almost always outweighs the modest latency cost.

### Server-side vs Client-side, summarized

A picture is worth a thousand words:

![Client-side vs Server-side discovery: comparison](../diagrams/svg/discovery-comparison.svg)

Both have legitimate uses. Modern best practice for most architectures: use **server-side discovery via the orchestrator** (Kubernetes Service, AWS ALB, etc.) as the default, and reach for client-side only when you have a specific reason (latency-critical, smart routing needs that the LB can't do).

### Service mesh as the synthesis

The service mesh ([Service Mesh](service-mesh.md)) has emerged as the dominant evolution that combines both views:

- From the app's perspective, the world is **server-side**: it calls a stable name (`orders-svc`), no discovery code needed.
- The local Envoy sidecar does **client-side LB** to choose among the actual pod IPs — zone-aware, latency-aware, with circuit breaking and retries.
- The control plane (Istio, Linkerd) keeps all the sidecars in sync with the registry.

Result: apps get server-side simplicity; the infrastructure gets client-side sophistication; both teams are happy. This is why mesh has become the de-facto answer for large polyglot architectures.

### What "server-side discovery" looks like in practice

In a Kubernetes cluster, a typical service-to-service call:

1. App calls `http://orders-svc/place`.
2. DNS resolves `orders-svc` (or `orders-svc.namespace.svc.cluster.local`) to a virtual IP.
3. The kernel's IP routing (managed by `kube-proxy` or eBPF) intercepts the connection and picks a backend pod from the Service's endpoint list. This is real-time; the endpoint list is updated as pods come/go.
4. The packet goes directly to the chosen pod.

There's no "load balancer" in the dedicated-process sense — the kernel itself is doing the LB based on a constantly-updated table. This is extremely fast and scalable. It's also why K8s discovery feels invisible: it's all in the kernel.

In a non-K8s cloud setup, the typical flow:

1. App calls `https://api.internal.company.com/place`.
2. DNS resolves to the AWS ALB's address.
3. ALB receives the connection; picks a target instance from its target group (synced from ECS or auto-scaling group).
4. ALB forwards. Caller sees ALB's response.

The ALB adds a real network hop (and a process boundary), but is a single concept that handles all discovery, health checks, and LB centrally.

### Pitfalls

- **DNS caching at the OS level** can serve stale entries longer than expected; ensure DNS TTL is short and clients respect it.
- **Connection pooling to the LB** can pin connections to the same backend even when new ones come online (LB sees a single connection, picks once). Solution: short connection lifetimes or HTTP/2 stream-level distribution.
- **LB latency in cold paths** — if the LB has its own startup costs (ALB target health checks take time on new deployments), make sure connection-establishment timeouts accommodate.
- **Hidden cross-AZ traffic** — server-side LBs typically don't preserve zone affinity by default; for cross-AZ cost-sensitive workloads, this matters.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Kubernetes Service (kube-proxy / IPVS / eBPF)** | The K8s built-in; the dominant server-side discovery today. |
| **Cloud LB (AWS ALB/NLB, GCP, Azure)** | Managed; auto-updates from the orchestrator. |
| **Self-managed NGINX/HAProxy** | Proxy with dynamic config from registry. |
| **Envoy at service edge** | Envoy as the per-service LB; building block for mesh. |
| **API Gateway as LB** | For north-south traffic; gateway routes to internal services. |
| **DNS-only "discovery"** | DNS-based with the LB transparent in DNS. Simplest. |
| **Mesh inbound sidecar** | Per-pod Envoy receiving traffic on behalf of the local app. |

## When NOT to use

- **When the LB hop is unacceptable** (latency-critical, low-level networking).
- **When the LB is a bottleneck** (extreme throughput requirements; mitigate with per-zone LBs).
- **When you need smart routing the LB can't do** — but consider mesh first before client-side library.
- **In environments without good cloud LBs or K8s** — bootstrapping discovery from scratch is more painful than client-side libraries.

---

## Real-world implementations

| System | Server-side discovery |
|---|---|
| **Kubernetes Service** | Default for K8s clusters. |
| **AWS ALB/NLB + Target Groups** | Cloud-managed; auto-syncs with ECS, EKS, ASG. |
| **GCP Internal Load Balancer** | Equivalent for GCP. |
| **Azure Load Balancer / Application Gateway** | Equivalent for Azure. |
| **NGINX Plus / HAProxy** | Dynamic config from Consul or via API. |
| **Consul + DNS** | Consul registers services; DNS resolves to LB. |
| **Envoy + control plane** | Mesh inbound model. |
| **Kong / Apigee / AWS API Gateway** | API gateway as discovery + routing. |
| **F5 BIG-IP** | Enterprise LB; widely used in financial services. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Every Kubernetes user** | K8s Service is server-side discovery; universally deployed. | ✅ Universal — K8s architecture |
| **AWS customers** | ALB + Target Groups is the dominant pattern for AWS-native architectures. | ✅ Verified — AWS docs |
| **GCP / Azure customers** | Equivalent managed LBs. | ✅ Verified — cloud docs |
| **Google internal** | GFE (Google Front End) is the canonical large-scale server-side LB. | ✅ Verified — multiple SRE book chapters |
| **HashiCorp Consul users** | Consul + DNS (or Connect mesh) for server-side discovery. | ✅ Verified — Consul case studies |
| **Many enterprise shops** | F5 + custom config-sync from registry. | ✅ Verified — F5 case studies |

---

## Further reading

- Chris Richardson, *microservices.io* — Server-Side Discovery pattern entry.
- Kubernetes documentation — Service, kube-proxy, IPVS, eBPF-based service routing.
- AWS documentation — ALB target groups, ECS service discovery integration.
- Newman, *Building Microservices* (2nd ed.) — Ch on inter-service communication.
- *Site Reliability Engineering* (Google) — Ch on load balancing at scale.
- Cilium documentation — eBPF-based service routing (modern K8s evolution).
- Buoyant / Solo.io blogs — mesh as evolution of server-side discovery.

---

*Diagram sources: [`../diagrams/src/server-side-discovery.d2`](../diagrams/src/server-side-discovery.d2), [`../diagrams/src/discovery-comparison.d2`](../diagrams/src/discovery-comparison.d2).*
