# Compute Resource Consolidation

**Aliases:** Compute Resource Consolidation, Bin Packing, Workload Consolidation, Multi-Tenancy at the Process/Container Level, Co-Location, Sidecar Stuffing (anti-pattern variant)
**Category:** Architecture / Operations
**Sources:**
[Microsoft Azure — Compute Resource Consolidation pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/compute-resource-consolidation) ·
[Kubernetes — Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) ·
[Borg paper — *Large-scale cluster management at Google with Borg* (Verma et al., 2015)](https://research.google/pubs/pub43438/) ·
[Mesos paper — *Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center* (Hindman et al., 2011)](https://www.usenix.org/legacy/event/nsdi11/tech/full_papers/Hindman_new.pdf) ·
*Building Microservices* (Newman, 2nd ed., 2021) — chapter on deployment ·
[AWS Fargate / Lambda documentation](https://aws.amazon.com/fargate/) — implicit consolidation

---

## Problem

> [!TIP]
> **ELI5.** If you give every tiny microservice its own dedicated VM, you end up with a fleet of mostly-empty servers (each at 5–15% CPU utilization) costing you full-VM prices. Plus every VM has its OS overhead, security patching, monitoring agents, certificate rotation — multiplied by N. **Compute resource consolidation** packs many workloads onto fewer compute units (via Kubernetes bin packing, multi-process containers, shared processes, or serverless platforms doing it transparently). You trade some isolation for huge cost and operational wins.

The natural microservices instinct — "one service, one deployment, one VM" — leads to wasted resources at scale. A typical small microservice uses a few hundred MB of RAM and a single-digit percentage of one CPU core most of the time. A VM that comfortably runs 50 such services costs the same as one running 1. The math:

- VM cost: $50/month for a small instance.
- 50 services, each on their own VM: $2,500/month.
- 50 services packed onto one VM: $50/month (if it fits).
- Even if you spread them across 3 VMs for redundancy and headroom: $150/month, vs $2,500. **A 16× cost saving.**

But cost isn't the only driver. Each separate VM also has:
- OS overhead (a few hundred MB just for the OS, agents, etc.).
- Patch management burden (50 OSes to patch vs 3).
- Monitoring agent overhead (50 agents reporting metrics).
- Security surface (50 SSH endpoints, 50 firewalls).
- Network overhead (each VM has its own NIC, IP, DNS entries).
- Slower deploys (50 deployment pipelines).

The pattern is to **consolidate** — pack many workloads into shared compute units — while preserving the right level of isolation. The art is choosing how much consolidation is appropriate for which workloads.

The pattern has a long history under different names:
- **Mainframe consolidation**: the original — one big machine, many workloads.
- **VM consolidation**: VMware, Xen, KVM packing many VMs onto fewer physical hosts.
- **Container consolidation**: Docker + Kubernetes packing many containers onto VMs.
- **Process consolidation**: many processes per container.
- **In-process plugins**: many "services" as modules in one process (Envoy filters, Nginx modules).
- **Serverless**: cloud platforms transparently consolidate functions onto shared fleets.

Each level represents a trade-off between density (cost) and isolation (safety).

## How it works

> [!TIP]
> **ELI5.** Stop giving every service its own dedicated server. Instead use an orchestrator (Kubernetes, Nomad) to pack many services onto fewer servers — like Tetris with workloads. Each service stays in its own container for isolation, but the underlying servers are well-utilized. For really small things (cron jobs, monitoring agents), combine them into one container. For really small services with the same lifecycle, share a process. Or use serverless platforms that do this transparently.

### The spectrum

The available consolidation approaches, from lightest to heaviest coupling:

![Compute resource consolidation](../diagrams/svg/compute-resource-consolidation.svg)

### 1. Bin packing via orchestrator (recommended default)

The Kubernetes / Nomad / Mesos approach. Each workload stays in its own container (process isolation, separate filesystem, cgroup limits). The orchestrator's **scheduler** packs many containers onto fewer nodes based on declared resource requests.

How it works:
- Each pod declares CPU/RAM requests and limits.
- Scheduler picks a node that has spare capacity.
- Multiple pods from different services share each node.
- Kernel enforces resource limits (cgroups).
- Node failure rolls workloads to other nodes.

Properties:
- **Isolation**: process, filesystem, network namespace per container.
- **Density**: high — typical nodes run 30–100+ pods.
- **Operational**: managed entirely by orchestrator; you don't directly manage which service is where.
- **Cost**: efficient — you pay for actual node count, not service count.

Pitfalls:
- Resource requests must be accurate; under-request causes contention, over-request wastes capacity.
- Default scheduling may not respect anti-affinity (HA needs explicit rules).
- Node-level failures take down multiple services (mitigated by spread).
- Noisy-neighbor effects across CPU and network.

This is the default for almost any new system at modest scale. Kubernetes pod density is the most common form of "compute resource consolidation" today.

### 2. Multi-workload container

Several related processes packed into one container, often with a supervisor (`supervisord`, `s6-overlay`, `runit`, `monit`).

Use cases:
- Utility containers (sidecar with multiple agents: log shipper + metrics + healthcheck).
- Legacy migrations (lift-and-shift of a multi-process app).
- Tiny cron jobs and daemons that don't justify their own pod.

Properties:
- **Less isolation**: processes share filesystem, network namespace.
- **Higher density**: one container instead of many.
- **Violates single-responsibility**: harder to scale processes independently.

This breaks the "one process per container" Docker guideline, and is generally an exception rather than a rule. Use sparingly; prefer Kubernetes pods with multiple containers when you need co-located processes with independent lifecycles.

### 3. Shared process / in-process plugins

The maximum-density option: many "services" live as modules/plugins inside one process. Examples:

- **Envoy filters**: HTTP filters compiled into the Envoy process.
- **Nginx modules**: Lua / Wasm modules in Nginx.
- **Service mesh data plane**: many services using a shared sidecar.
- **Database extensions**: Postgres extensions running in-process.
- **WordPress plugins**: many "features" in one PHP process.
- **VS Code extensions**: extensions in the same renderer/host process.

Properties:
- **Highest density**: shared OS, shared runtime, shared memory.
- **Lowest isolation**: a buggy plugin crashes everyone.
- **Tight coupling**: deploy any plugin = deploy the whole thing.
- **Shared trust boundary**: only safe if all plugins are trusted.

Used widely but specifically — usually where the host platform was designed for it and the lifecycle of plugins is acceptable.

### 4. Serverless (transparent consolidation)

AWS Lambda, Google Cloud Functions, Azure Functions, Cloud Run, Fargate: the platform handles consolidation. You write your function or container; the platform decides how to pack invocations onto its fleet. You pay only for the time your code runs.

Properties:
- **Transparent**: you don't see the underlying packing.
- **Pay-per-use**: idle time costs nothing.
- **Limits**: cold starts, max execution time, memory ceilings.
- **Lock-in**: vendor-specific runtimes (mostly).

For event-driven or low-traffic workloads, this is often the cheapest and operationally simplest option — the cloud provider does the consolidation that Kubernetes would otherwise require you to do.

### When to consolidate (and when not to)

**Consolidate when**:
- Services have similar SLAs (you can scale them together).
- Services have complementary load patterns (peaks don't all collide).
- Services share trust boundary (don't co-locate untrusted code).
- Operational overhead per service is high.
- Cost-sensitive workloads (small companies, dev/test environments).
- Many small utility services.

**Don't consolidate when**:
- Services have very different scaling needs (one needs 50 replicas, another needs 1).
- Different security / compliance domains (PCI scope, HIPAA).
- Noisy-neighbor risk is unacceptable (latency-critical service can't share with a batch job).
- Different language / runtime constraints (one needs Java 21, another needs Java 8).
- Fault isolation matters more than density (per-tenant isolation in multi-tenant SaaS).
- Resource patterns are unpredictable.

### The noisy neighbor problem

The classic risk of consolidation: one workload spikes (CPU, memory, network, disk I/O) and starves co-tenants. Mitigations:

- **Resource limits** (cgroups in Linux, cgroup v2 better than v1): hard cap CPU and memory.
- **QoS classes** (Kubernetes Guaranteed > Burstable > BestEffort): scheduler treats them differently under pressure.
- **Dedicated nodes for noisy workloads**: explicit anti-co-location.
- **Bandwidth shaping** (tc qdisc): limit network use.
- **Block I/O limits** (cgroups, ionice).
- **Pod anti-affinity rules**: don't put X and Y on the same node.

The maturity of cgroup v2 + Kubernetes resource management has made noisy-neighbor problems much more tractable than they were on raw VMs a decade ago — but they're never fully eliminated, especially at the CPU cache, memory bandwidth, and shared-disk levels.

### The fault blast radius problem

When you pack 20 services on one VM and the VM dies, you lose 20 services. Mitigations:

- **Spread**: schedule replicas across multiple nodes (Kubernetes pod anti-affinity, AZ spread).
- **Multi-AZ packing**: don't pack so densely that a node loss saturates surviving nodes.
- **[Static stability](../res/static-stability.md)**: maintain spare capacity so a node loss is absorbed.
- **Health checks**: detect and reschedule fast.

Combined with **[Bulkhead](../res/bulkhead.md)** thinking: consolidate within bulkheads, not across them. Co-locate services that share a fate; isolate services with different SLAs.

### Multi-tenancy as consolidation

A related pattern: **multi-tenant SaaS** is consolidation taken further — many *customers* share the same compute (and often the same database tables, with row-level isolation). This is consolidation in the limit:

- **Pool model**: one set of compute resources serves all tenants. Cheapest, hardest to isolate.
- **Silo model**: dedicated compute per tenant. Most isolation, most expensive.
- **Bridge / hybrid**: shared compute for small tenants; dedicated for large.

Most modern SaaS uses pool with strong app-layer isolation. AWS, Salesforce, Slack, Shopify all run massive consolidated pools with multi-tenant tables.

### Anti-patterns

- **"Microservices on dedicated VMs" by default**: wastes money, increases ops overhead.
- **No resource requests / limits in Kubernetes**: scheduler can't pack well; risks noisy-neighbor chaos.
- **Massively oversized resource requests**: cluster utilization plummets.
- **Combining services with very different SLAs**: the strict one suffers from the lax one's neighbors.
- **Combining trusted and untrusted code in one container**: classic privilege-escalation risk.
- **Over-stuffing containers with `supervisord`**: hides operational complexity; harder to debug.
- **Ignoring noisy neighbors**: blame the wrong thing during incidents.

### Trade-offs

Advantages:
- **Lower cost** (often 5–20× cheaper at scale).
- **Lower operational overhead** (fewer hosts/agents/patches).
- **Better resource utilization** (CPUs/RAM not idle).
- **Faster deploys** (smaller fleet).
- **Greener** (less wasted compute).
- **Modern orchestrators handle complexity**.

Disadvantages:
- **Reduced isolation** (vs dedicated VMs).
- **Noisy neighbor risk** (mitigated by limits).
- **Fault blast radius** (one node = many services).
- **Coupled lifecycles** at higher consolidation levels.
- **Observability harder** (attribute usage per service).
- **Capacity planning more complex** (must reason about mix).

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Compute Resource Consolidation (Azure)** | The named pattern. |
| **Bin packing** | Scheduler-level density. |
| **Multi-tenancy** | Customers (not services) share resources. |
| **Co-tenancy / shuffle sharding** | Per-tenant grouping for partial isolation. |
| **Serverless** | Provider-managed consolidation. |
| **Mainframe / VM consolidation** | Historical ancestors. |
| **Container orchestration** | Modern enabler (Kubernetes, Nomad, ECS). |
| **[Sidecar](../comm/sidecar.md)** | Co-located helper container; specific consolidation form. |
| **Service mesh** | Single data-plane sidecar serving many flows. |
| **[Static Stability](../res/static-stability.md)** | Pairs with consolidation: don't pack to 100%. |
| **[Bulkhead](../res/bulkhead.md)** | Counter-pattern: isolation over density when needed. |
| **[Deployment Stamps](../ops/deployment-stamps.md)** | Stamps may share or dedicate compute. |

## When NOT to use

- Workloads in different compliance / security domains.
- Latency-critical services co-located with batch workloads.
- Services with very different scaling profiles.
- Where strict fault isolation is required.
- Very low-volume environments where overhead doesn't matter.
- Tiny systems where simplicity > efficiency.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **Kubernetes** | Bin-packing orchestrator (de facto standard) |
| **Nomad** | Alternative orchestrator |
| **Apache Mesos + Marathon** | Earlier orchestrator |
| **AWS ECS / Fargate** | AWS-managed |
| **Azure Container Apps / AKS** | Azure-managed |
| **GCP Cloud Run / GKE** | GCP-managed |
| **AWS Lambda, GCP Cloud Functions, Azure Functions** | Serverless consolidation |
| **Docker Compose** | Local consolidation |
| **supervisord, s6-overlay, runit** | Multi-process containers |
| **Envoy, Nginx with modules** | In-process plugin consolidation |
| **Borg (Google), Twine (Meta), Apollo (Microsoft)** | Internal orchestrators |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Google** | Borg packs all of Google's workloads onto shared clusters. | ✅ Verified — Borg paper (2015) |
| **Meta / Facebook** | Twine cluster manager; massive consolidation. | ✅ Verified — Twine paper (OSDI 2020) |
| **Twitter** | Mesos at scale (historical), now Kubernetes. | ✅ Twitter engineering blog |
| **Airbnb** | Heavy Kubernetes consolidation. | ✅ Airbnb engineering blog |
| **Spotify** | Backstage + Kubernetes for service density. | ✅ Spotify engineering posts |
| **AWS** | Fargate / Lambda are commercialized consolidation. | ✅ AWS product docs |
| **Most cloud-native companies (2018+)** | Default to Kubernetes bin packing. | ✅ Industry universal |
| **Salesforce, Shopify, Slack** | Multi-tenant SaaS = consolidation at customer level. | ✅ Documented in talks |

---

## Further reading

- Microsoft Azure — Compute Resource Consolidation pattern.
- *Large-scale cluster management at Google with Borg* (Verma et al., EuroSys 2015).
- *Twine: A Unified Cluster Management System* (Meta, OSDI 2020).
- *Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center* (Hindman et al., NSDI 2011).
- Kubernetes documentation — resource management, scheduler, QoS classes.
- *Building Microservices* (Newman, 2nd ed.) — Ch on deployment.
- *Kubernetes Patterns* (Ibryam & Huß, O'Reilly 2nd ed., 2023).
- *Designing Distributed Systems* (Burns, 2018).
- AWS Fargate / Lambda whitepapers.
- *Cloud Native Patterns* (Davis, Manning 2019).

---

*Diagram source: [`../diagrams/src/compute-resource-consolidation.d2`](../diagrams/src/compute-resource-consolidation.d2).*
