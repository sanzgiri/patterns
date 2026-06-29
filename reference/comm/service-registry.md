# Service Registry

**Aliases:** Service Catalog, Service Directory, Naming Service, Endpoint Registry
**Category:** Service discovery
**Sources:**
[Chris Richardson — microservices.io: Service Registry](https://microservices.io/patterns/service-registry.html) ·
[Microsoft Azure Architecture Center — Service Registry](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/interservice-communication) ·
Discussion in Newman, *Building Microservices* (2nd ed.)

---

## Problem

> [!TIP]
> **ELI5.** In a microservices system, instances come and go all the time — autoscaling adds and removes pods, deployments roll new versions in, hosts die. How does a caller find the *currently-running healthy* instances of a service? Hard-coding IP addresses doesn't work. You need a **service registry** — a live, up-to-date directory of "what's running where," that callers can query.

A microservices architecture has hundreds or thousands of service instances that:

- **Come and go dynamically.** Autoscaling adds capacity in seconds; spot instances disappear; deployments roll new versions across the fleet; nodes fail.
- **Live on ephemeral addresses.** Container IPs change on restart; pods get new addresses; cloud VMs have new private IPs after replacement.
- **Have varying health.** A given instance might be alive but failing health checks (degraded), or starting up (not ready), or shutting down (draining).
- **Span regions and zones.** Production deployments have instances in multiple availability zones, regions, or data centers.

A caller needs to know: "what's the current set of healthy `orders-svc` instances I can call?" The naive answer — hardcoding addresses or using a static config file — fails immediately in a real cluster. Static DNS gives you names but not freshness or health awareness.

The **service registry** is the source of truth: a queryable, watch-able directory mapping service names to current instance addresses and health states. Every service-discovery scheme (client-side, server-side, mesh-based) ultimately reads from a registry.

## How it works

> [!TIP]
> **ELI5.** Build (or pick) one piece of infrastructure that knows where every service instance lives. Services register themselves when they start, send periodic heartbeats to prove they're alive, and deregister when they shut down (or get evicted if heartbeats stop). Callers query the registry to find healthy instances. The registry is typically replicated for high availability and supports watching for live updates.

A service registry stores, for each service:

![Service Registry: structure and flow](../diagrams/svg/service-registry.svg)

- The **service name** — a stable logical identifier (`orders-svc`, `payment-api`).
- A list of **instance records**, each containing:
  - host:port (or pod IP, or DNS name).
  - **Health status** — typically alive/sick/dead, derived from heartbeats and/or active health checks.
  - **Metadata** — version, region, availability zone, environment, tags.
  - Last heartbeat timestamp.

The registry exposes:

- **Read API**: "give me all healthy instances of `orders-svc` in this region."
- **Write API**: register, heartbeat, deregister.
- **Watch API**: stream updates as the set of instances changes (so callers don't poll).
- **Health checks**: either active (registry pings instances) or passive (instances report).

Underneath, the registry itself is a small distributed system — typically built on Raft or ZooKeeper consensus for high availability and consistency. Eureka, Consul, etcd, ZooKeeper, and Kubernetes' built-in etcd-backed endpoints all serve this role.

### Registration patterns: who tells the registry?

Two ways an instance gets into the registry:

![Self-registration vs Third-party registration](../diagrams/svg/registration-patterns.svg)

**Self-registration**: the service itself, on startup, calls the registry to announce its existence. Periodically sends heartbeats. Calls the registry on shutdown to deregister.

Pros: simple in homogeneous environments; the service has direct control.

Cons: every service in every language needs a registration client; services can forget to deregister (causing stale entries, partly cured by heartbeat timeout); registration logic clutters service code.

Used by: Eureka clients (Netflix OSS), Consul agent (when configured for self-registration), Spring Cloud apps.

**Third-party registration**: a separate component — typically the orchestrator itself or an agent — watches deployment events and registers/deregisters on the service's behalf.

Pros: service code knows nothing about registration; polyglot-friendly; orchestrator visibility ensures deregistration on shutdown; loose coupling between service and registry.

Cons: extra moving piece; registrar is a new dependency for correctness.

Used by: **Kubernetes** (the endpoints controller watches pod lifecycle and updates Service endpoints automatically), AWS ECS service-discovery integration, Nomad's built-in registration.

In practice, **third-party registration via the orchestrator has won** in modern cloud-native deployments. Kubernetes handles it transparently — you define a Service, and the registry (etcd) is kept in sync without any application code. Self-registration is mostly a legacy of pre-orchestrator microservices.

### Health checks

The registry's value depends on accurate health information. Two approaches:

- **Heartbeats**: the instance periodically tells the registry "I'm alive." If the registry stops hearing for some interval, it marks the instance dead. Simple but coarse — a heartbeat-alive instance can still be returning 500s.
- **Active health checks**: the registry (or another component) periodically calls a `/health` endpoint on each instance. More accurate but requires the registry to reach every instance, which is hard in some network topologies.

Most production registries do both: heartbeats for liveness ("the process is up"), plus a health check the registry validates ("the application is functioning").

### Replication and HA

The registry is critical infrastructure. If it goes down, no service can find other services — and the system collapses. So:

- The registry is typically **replicated using consensus** (Raft for etcd/Consul; ZAB for ZooKeeper). Survives minority node failures.
- Callers often **cache** the last known registry state locally and serve from cache during brief registry outages.
- Some clients **fall back to DNS** or static config if the registry is unreachable.
- The registry is often **partitioned regionally** so a region's registry outage doesn't take down other regions.

These mechanisms make the registry's effective availability much higher than a naive "always call the registry" design would imply.

### Registry implementations

The space is mature with multiple solid choices:

- **Eureka** (Netflix OSS, now community-maintained): AP-favoring (chooses availability over consistency in CAP terms); good for high-churn environments where stale-but-available data beats consistent-but-unavailable.
- **Consul** (HashiCorp): CP-favoring with gossip-based health, integrates with mesh, multi-DC native.
- **etcd**: CP, used internally by Kubernetes; rarely accessed directly as a registry.
- **ZooKeeper**: CP, the original; still used by Kafka, Hadoop, older systems.
- **Kubernetes Service + endpoints**: built-in registry for K8s clusters; the dominant choice when you're on K8s.
- **AWS Cloud Map**: managed registry on AWS; integrates with ALB, ECS, EKS.
- **DNS with SRV records**: simple, universal; less fresh than purpose-built registries.

The choice depends on your environment: on Kubernetes, use the built-in. On AWS without K8s, Cloud Map or Consul. On-prem polyglot environments, Consul. Legacy Java-heavy Netflix-style setup, Eureka.

### Common pitfalls

- **No deregistration on shutdown** → stale entries → callers route to dead instances.
- **Health checks too lenient** → sick instances stay in rotation, causing latency spikes.
- **Health checks too aggressive** → flapping instances; constant routing churn.
- **Heartbeat interval too long** → slow detection of failures.
- **Heartbeat interval too short** → registry overloaded by heartbeat traffic at scale.
- **Single registry for everything** → registry becomes the bottleneck; consider sharding by service or region.
- **Callers don't cache** → registry outage takes down the system.

The good registry implementations have sensible defaults for most of these, but real production systems still tune them based on traffic patterns.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **In-band (DNS-based)** | Registry queryable via standard DNS (Consul, K8s DNS, AWS Cloud Map). Universal protocol. |
| **Library-based** | Eureka-style: clients use a specific library to query. |
| **API-based** | Custom HTTP/gRPC API; consumed by sidecars or apps. |
| **Self-registration** | Service registers itself. |
| **Third-party registration** | Orchestrator or agent registers on service's behalf. |
| **Active health checks** | Registry probes instances. |
| **Passive health checks** | Instances heartbeat. |
| **Multi-region registry** | Federated registries across regions. |
| **Mesh-integrated** | Consul + Connect; Kubernetes + Istio — registry pre-integrated with mesh. |

## When NOT to use

- **Single static service** with one well-known endpoint — DNS is enough.
- **Trivial deployments** with hardcoded IPs — small experimental setups.
- **Environments where the orchestrator already provides discovery** — don't add a redundant registry on top of K8s Service.
- **No need for health-aware routing** — basic DNS resolution suffices.

---

## Real-world implementations

| Registry | Notes |
|---|---|
| **Kubernetes Service + endpoints** | Built into every K8s cluster; the default for K8s deployments. |
| **Consul** | HashiCorp; multi-DC native, integrates with Vault and Connect mesh. |
| **etcd** | Foundation for K8s; rarely used directly as a service registry. |
| **ZooKeeper** | Legacy; underpins Kafka, Hadoop, older systems. |
| **Eureka** | Netflix OSS, AP-favoring; once dominant in Spring Cloud apps. |
| **AWS Cloud Map** | Managed registry for ECS, EKS, Lambda. |
| **Google Cloud Service Directory** | Managed registry on GCP. |
| **NATS Service** | NATS messaging system has built-in service framework. |
| **CoreDNS plugins** | DNS-based discovery for various backends. |
| **Apache Curator / DiscoveryClient** | Service-discovery on ZooKeeper. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Netflix** | Built Eureka to handle 100k+ instances in the early 2010s; now uses K8s-based discovery in newer systems. | ✅ Verified — [Eureka on GitHub](https://github.com/Netflix/eureka) |
| **Every Kubernetes user** | K8s' built-in Service + endpoints is the registry for all K8s deployments. | ✅ Universal — K8s architecture |
| **HashiCorp customers (every Consul user)** | Consul as registry across many enterprises (Cisco, Comcast, etc.). | ✅ Verified — Consul case studies |
| **Apache Kafka, Hadoop, HBase** | ZooKeeper is the registry for these foundational systems. | ✅ Verified — project documentation |
| **AWS customers using ECS / App Mesh** | Cloud Map as registry. | ✅ Verified — AWS docs |
| **Airbnb, Yelp, eBay** | Historically used Eureka or Consul; now mixed with K8s-native. | ⚠ Mixed; specific architecture has evolved |

---

## Further reading

- Chris Richardson, *microservices.io* — Service Registry pattern entry.
- Newman, *Building Microservices* (2nd ed., 2021) — Ch on inter-service communication.
- Kubernetes documentation — Service, endpoints, and EndpointSlice resources.
- Consul documentation — service discovery section.
- *Designing Distributed Systems*, Brendan Burns — Ch on patterns including discovery.
- Netflix Tech Blog — historical posts on Eureka design and trade-offs.
- HashiCorp's "Consul vs Eureka vs ZooKeeper" comparison.

---

*Diagram sources: [`../diagrams/src/service-registry.d2`](../diagrams/src/service-registry.d2), [`../diagrams/src/registration-patterns.d2`](../diagrams/src/registration-patterns.d2).*
