# Static Stability

**Aliases:** Static Stability, Statically Stable Architecture, Zonal Independence, Pre-Provisioned Failover, Data-Plane Isolation
**Category:** Resilience / Architecture
**Sources:**
[Amazon Builders' Library — *Static stability using Availability Zones* (Becky Weiss)](https://aws.amazon.com/builders-library/static-stability-using-availability-zones/) — primary source ·
[Amazon Builders' Library — *Avoiding fallback in distributed systems*](https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/) ·
[AWS re:Invent 2018 — *How AWS Minimizes the Blast Radius of Failures* (ARC338)](https://www.youtube.com/watch?v=swQbA4zub20) ·
[AWS re:Invent 2022 — *Building resilient multi-Region applications* (ARC209)](https://www.youtube.com/results?search_query=arc209+reinvent+2022) ·
[*Site Reliability Engineering* (Google, 2016) — Ch 22, Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) — related principle

---

## Problem

> [!TIP]
> **ELI5.** When something fails in your system, the obvious response is "do something about it!" — failover, scale up, reroute, push new config. But that **reactive action during a failure is itself dangerous**: the systems you'd use to do it might also be failing; the action might cascade load to the survivors; the orchestration might depend on the very thing that broke. **Static stability** flips the script: pre-provision capacity, pre-distribute data, and arrange things so that when something fails, **doing nothing** keeps the system working. Spend money on idle headroom; spend resilience budget on architecture; don't bet your availability on perfect orchestration during chaos.

Most outages aren't caused by the original fault — they're caused by the *reaction* to the fault. Classic examples from public postmortems:

- **AWS S3 outage Feb 28, 2017**: a single typo took down S3 in `us-east-1`. The recovery took 4 hours because the operations tools for restarting subsystems required S3 to be up. Static stability principle violated: the recovery path depended on the failed system.
- **Facebook BGP outage Oct 4, 2021**: a routing config push isolated Facebook from the internet for 6+ hours. The badge readers and internal tools to fix it were also offline. Even physical access to data centers was impeded.
- **Google Cloud outage June 2, 2019**: configuration change disabled network capacity across multiple regions. The control plane that would have rolled it back was also affected.
- **Many DB failover incidents**: reactive failover takes longer than expected; old primary returns thinking it's still primary; split-brain.

The pattern that addresses this is **static stability** (named by Becky Weiss / AWS, c. 2018): design so that **the system stays in its last-known-good state when a dependency fails**. The system doesn't try to do anything fancy. It just keeps doing what it was already doing. The failure response was baked in at provisioning time, not invented during the incident.

The classic example: AWS recommends **active-active across 3 AZs at 50% capacity each, sized so any 2 AZs can serve all traffic**. When an AZ fails, the surviving 2 absorb the load without any action — DNS health checks gradually shift, no orchestration runs, no config push, no scaling event. The system is *statically stable* under single-AZ loss.

Compare to the more "natural" approach of running 3 AZs at 33% each and scaling up the survivors when one fails: that response requires the control plane, takes time, and amplifies failure (the survivors are suddenly at 50% and might be overwhelmed before scaling completes).

The principle generalizes well beyond AZs:

- Multi-region: pre-provision capacity in N regions sized for N-1 to carry load.
- Caches: pre-warmed, not cold-start on miss.
- Replicas: hot/synchronous, not promote-on-failure.
- Config: cached locally with last-known-good; not refetched on every request.
- Service discovery: cached endpoints; tolerate registry outage.
- Auth tokens: pre-issued, cached; tolerate brief IdP outage.

The cost is real: idle capacity is wasted money. But the alternative — outages — is usually more expensive in money, reputation, and team morale.

## How it works

> [!TIP]
> **ELI5.** Build the system as if a failure has already happened — extra capacity, replicated data, parallel paths — so when one really happens, the system just keeps running on what's still healthy. Don't try to be clever during a fire. Be lazy. Lazy is safer.

### The picture

The contrast between dynamic and static approaches:

![Static stability](../diagrams/svg/static-stability.svg)

### The four core practices

**1. Over-provision for failure.** Run capacity sized so the loss of one failure unit (AZ, region, replica) is absorbed by survivors without new resource acquisition. Concrete:

- 3 AZs at 50% capacity each → lose 1, survivors run at 75% (within their headroom).
- 2 AZs at 100% each → lose 1, other carries full load.
- 4 AZs at 33% → lose 1, survivors at 44% (cheap; common for very cost-sensitive).

The exact number depends on cost vs availability trade-off. AWS publicly recommends the 3-AZ pattern as default.

**2. Pre-distribute state and config.** All data needed to operate during failure is *already in place* before the failure. No live cross-AZ or cross-region calls in the hot path during failure.

- Database replicas in each AZ, with synchronous or near-synchronous replication.
- S3 / object-store cross-region replication so reads are local.
- Configuration distributed via push (not pulled on demand from a central service).
- DNS records already point to multi-AZ load balancers; health checks shift weights without manual intervention.

**3. Stay in known-good state during failure.** When a failure happens, **do nothing new**. Just let the failed component drop out. Surviving components are already configured to serve the load. No orchestration runs. No human decisions in the critical path.

- Health checks fail; load balancer stops sending traffic to bad instances.
- DNS TTL expires; clients resolve to surviving endpoints.
- Replicated services continue serving.
- Nothing is freshly created, freshly configured, freshly scaled.

**4. Data plane independent of control plane.** The serving path (data plane) must not depend on the management path (control plane) during failures. Examples:

- EC2 instances continue running and serving traffic even when the EC2 API is degraded (you can't start new instances, but existing ones keep working).
- S3 keeps serving GET/PUT even when bucket creation is impaired.
- DNS lookups (data plane) work even when DNS administration (control plane) is unavailable.
- Cached IAM permissions continue authorizing even when IAM is briefly unavailable.

The general statement: **the system continues to work in the absence of new control-plane actions**. Control planes (which are the most complex part of any system) should not be in the critical path of serving traffic.

### Why "stay in known-good" beats "react smartly"

Reactive failover seems obvious but fails for several reasons:

- **Control-plane dependency**: the failover tool often depends on the failed system.
- **Detection lag**: by the time you detect, react, and roll out, the user has already noticed.
- **Cascading load**: shifting load to survivors can overwhelm them (thundering herd; see [page](thundering-herd.md)).
- **Race conditions**: split-brain when failed system returns thinking it's primary.
- **Untested code paths**: failover logic runs rarely; bugs hide until needed.
- **Human cognition under stress**: incident response is the worst time to make decisions.

Static stability replaces this with: pre-provisioned, always-on capacity that just absorbs the loss. The "failover" code is now zero code; it's just "DNS routes around the dead host" or "load balancer stops sending here." Battle-tested by happening continuously, not annually.

### Concrete AWS implementations

- **Application Load Balancer + Auto Scaling Group across 3 AZs**: ALB health-checks per AZ; ASG keeps the count balanced; no orchestration needed for AZ loss.
- **RDS Multi-AZ**: synchronous standby in another AZ; promoted automatically; same DNS endpoint.
- **DynamoDB**: replicated within region across AZs by default; multi-region tables for region loss.
- **S3**: replicated across AZs within region; cross-region replication available.
- **Route 53 health checks + DNS failover**: pre-configured DNS records with health-check-controlled weights.
- **Lambda + provisioned concurrency**: warm capacity, not cold-start during failure.

### Beyond AWS

The principle applies anywhere:

- **Kubernetes**: pre-provisioned pod counts across nodes; static affinity rules. Pre-pulled container images. Don't pull on demand during an incident.
- **GCP**: regional managed instance groups with autohealing; multi-region load balancers.
- **Azure**: availability sets, zone-redundant storage, Traffic Manager.
- **Multi-cloud**: hot/hot setups (rare; expensive; but possible).

### Trade-offs

Advantages:
- **Reliability**: failures are absorbed without new code paths firing.
- **Predictable behavior**: behavior during failure is the same as behavior under load.
- **Simpler recovery**: less to coordinate during incidents.
- **Tested by normal operation**: the "failover" path runs every day at lower volume.
- **Independent of control-plane health**: serving continues during control-plane outages.

Disadvantages:
- **Cost**: idle capacity costs money continuously, not just during incidents.
- **Capacity planning is harder**: you're sizing for the failure case.
- **Wastes resources in steady state**: 50% AZ utilization means you bought twice the hardware.
- **Doesn't address every failure mode**: a logic bug deployed everywhere still breaks everything statically stably.
- **Can mask underlying degradation**: if you always run at 50%, you might not notice creeping load until you're at 99% during the next failure.

### Combining with other patterns

Static stability composes naturally with:

- **[Bulkhead](bulkhead.md)**: isolate failures so they don't spread across cells.
- **[Circuit Breaker](circuit-breaker.md)**: stop sending to broken dependencies; survive their loss.
- **[Backpressure](backpressure.md)**: shed load when overloaded; preserve known-good service to remaining clients.
- **[Deployment Stamps / Geode](../ops/deployment-stamps.md)**: stamps are statically stable units.
- **[Chaos Engineering](../ops/chaos-engineering.md)**: continuously verify that the statically-stable design actually survives failures.
- **Cache-aside / pre-warmed caches**: pre-loaded data is statically stable; cold caches aren't.

### When NOT to use

- **Cost-sensitive workloads** where reactive failover is acceptable.
- **Very predictable, low-variance load** where idle capacity is genuinely wasted.
- **Stateless workloads** where spin-up is fast enough.
- **Development / test environments**.
- **Where reactive failover is mature and well-tested** (and you accept the cost of testing it continuously).

### Common anti-patterns it forbids

- **Cold-standby DR with manual failover**: violates static stability — the failover code path is the riskiest part.
- **Reactive autoscaling as the primary failure response**: scaling takes time; failures don't wait.
- **Fetching config on every request from a central service**: central service outage propagates immediately.
- **DNS TTL of 60 seconds with no pre-distributed records**: 60s is forever during a fire.
- **Failover that requires multiple services to coordinate**: each is another failure surface.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Static Stability (Builders' Library)** | The canonical AWS name. |
| **Zonal independence** | Same principle, scoped to AZs. |
| **Cellular isolation** | Cells are statically stable units. |
| **Pre-warmed capacity** | Specific tactic. |
| **Read replicas with sync replication** | Pre-promoted vs reactive failover. |
| **Active-Active multi-region** | Static stability extended to regions. |
| **[Bulkhead](bulkhead.md)** | Isolation enabler. |
| **[Circuit Breaker](circuit-breaker.md)** | Reactive complement (still useful). |
| **[Backpressure](backpressure.md)** | Reactive load-shedding complement. |
| **[Deployment Stamps](../ops/deployment-stamps.md)** | Stamps as statically stable units. |
| **[Chaos Engineering](../ops/chaos-engineering.md)** | Verifies the design holds. |

## When NOT to use

- Truly cost-constrained systems.
- Workloads where degraded availability during failure is acceptable.
- Where reactive failover is well-tested via continuous chaos testing.
- Internal tools with low availability SLOs.

---

## Real-world implementations

| Tool / Service | Statically stable design |
|---|---|
| **AWS ALB + ASG multi-AZ** | Built-in static stability |
| **AWS RDS Multi-AZ, Aurora Multi-AZ** | Sync standby |
| **AWS DynamoDB / S3** | Replicated within region; cross-region available |
| **AWS Route 53 health checks** | DNS-level failover without orchestration |
| **GCP regional MIGs + GLB** | Equivalent design |
| **Azure availability sets + zone-redundant storage** | Equivalent design |
| **Kubernetes anti-affinity + multi-zone clusters** | DIY static stability |
| **Pre-warmed Lambda / Cloud Run** | Provisioned concurrency |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **AWS** | Coined the term; pervasive internal design principle. | ✅ Verified — Builders' Library |
| **Netflix** | Multi-region active-active; absorbs region loss. | ✅ Verified — Netflix Tech Blog |
| **Stripe** | Multi-region capacity headroom. | ✅ Mentioned in talks |
| **Slack** | Cell-based static stability per shard. | ✅ Slack engineering posts |
| **Most large AWS customers** | Apply Builders' Library guidance. | ✅ Industry common |
| **Google SRE** | Equivalent principles in *SRE Book* Ch 22. | ✅ Verified — SRE book |
| **Cloudflare** | Anycast + multi-PoP architecture is statically stable. | ✅ Verified — Cloudflare blog |
| **Banks / regulated industries** | Heavy investment in statically stable multi-DC. | ✅ Industry standard |

---

## Further reading

- Amazon Builders' Library — *Static stability using Availability Zones* (Weiss).
- Amazon Builders' Library — *Avoiding fallback in distributed systems*.
- AWS re:Invent ARC338 (2018) — *Minimizing the blast radius of failures*.
- *Site Reliability Engineering* (Google, 2016) — Ch 22.
- *Release It!* (Nygard, 2018, 2nd ed.) — stability patterns.
- *Chaos Engineering* (Rosenthal & Jones, O'Reilly 2020).
- *Building Microservices* (Newman, 2nd ed.) — Ch 12.
- AWS Well-Architected Framework — Reliability pillar.
- Public postmortems: AWS S3 (Feb 2017), Facebook BGP (Oct 2021), Google Cloud (June 2019).

---

*Diagram source: [`../diagrams/src/static-stability.d2`](../diagrams/src/static-stability.d2).*
