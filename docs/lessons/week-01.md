# Week 1 — Requirements, boundaries, and scaling shape

## Outcome

By the end of this week, you should be able to turn an ambiguous product request into explicit system requirements, estimate the dominant scale, choose an initial architecture, and explain why a monolith may be the correct first boundary.

## Patterns and material

- [Monolith](../../reference/arch/monolith.md)
- [Service Decomposition](../../reference/arch/service-decomposition.md)
- [Microservices](../../reference/arch/microservices.md)
- [CDN](../../reference/scale/cdn.md)
- [Deployment Stamps](../../reference/ops/deployment-stamps.md)

## Session 1 — Requirements are design inputs

Start with five dimensions:

1. **Users and actions:** who uploads, views, searches, shares, or deletes?
2. **Scale:** daily active users, peak requests per second, object sizes, growth rate.
3. **Performance:** p50/p95 latency and which actions are interactive.
4. **Correctness:** what must be immediately correct versus eventually correct?
5. **Failure and policy:** durability, privacy, regional residency, abuse, and recovery objectives.

The key distinction is between a product requirement and a system guarantee. “Users see their photos” is a product requirement. “An acknowledged upload is not lost after a single-region failure” is a guarantee that drives storage and replication.

### Exercise

For a photo-sharing service, write ten assumptions before drawing any boxes. Include 100 million daily users, 10 photos uploaded per active user per month, 5 MB average original size, 20× read-to-write ratio, and a global audience. Mark each assumption as a number to verify, a hard requirement, or a negotiable choice.

### Checkpoint

You are ready to continue when you can answer: “What is the first bottleneck, and what evidence would prove you wrong?”

## Session 2 — Back-of-the-envelope capacity

Use rough numbers to identify shape, not to claim precision. For the exercise:

- uploads/month ≈ 100M × 10 = 1B;
- average uploads/second ≈ 1B / 2.6M seconds ≈ 385;
- design peak at 10× average ≈ 4K uploads/second;
- if each upload is 5 MB, raw monthly ingress ≈ 5 PB;
- reads are much larger, so image delivery—not metadata writes—is the first obvious bandwidth concern.

This immediately suggests separating binary-object delivery from metadata APIs. It does not yet justify microservices. A relational metadata store, object storage, asynchronous thumbnail workers, and a CDN may be enough.

### Design heuristic

Partition by workload before partitioning by team. Photos, metadata, feeds, search, and notifications have different access patterns. A boundary is valuable when it isolates a scaling, reliability, ownership, or data-model difference—not merely because a noun exists in the domain.

## Session 3 — Monolith first, boundaries later

An initial architecture:

```text
client → API service → metadata database
                  ↘ object-storage upload URL
object storage → event/queue → thumbnail and moderation workers
client ← CDN ← object storage
```

Keep user, photo metadata, permissions, and sharing transactions together initially. Use a direct upload URL so large bytes bypass the application tier. Make thumbnail generation asynchronous because it is derived work.

Possible future boundaries:

- media processing, because it is CPU/GPU-heavy and asynchronous;
- feed generation, because fan-out creates a different scaling shape;
- search, because it owns an index and tolerates eventual consistency;
- identity, if shared across many products or governed separately.

Avoid splitting a service while its data and transaction boundary are still unclear. A distributed transaction is often the cost of a premature boundary.

## Session 4 — Interview case

### Prompt

“Design a global photo-sharing service.” You have 35 minutes.

### Strong answer outline

1. Clarify upload, view, share, delete, privacy, and feed requirements.
2. Estimate metadata QPS, object storage, egress, and peak traffic.
3. Separate control plane (metadata and permissions) from data plane (photo bytes).
4. Use object storage plus presigned upload/download URLs and a CDN.
5. Make thumbnails and moderation event-driven and idempotent.
6. State consistency: an owner’s delete should invalidate access promptly; public feed propagation can be eventual.
7. Identify observability: upload success, processing lag, CDN hit rate, origin egress, stale-access incidents.
8. Explain the first scale-out boundary and why it is not “microservices everywhere.”

### Requirement change

“A celebrity posts a photo and 30 million users request it in one minute.” Add CDN caching, request coalescing, origin protection, and a strategy for invalidation. The important insight is that the hot object is a delivery problem, not a database-sharding problem.

## Deliverable and rubric

Submit one page with an architecture diagram, assumptions, capacity estimates, consistency choices, first failure mode, and first decomposition boundary.

Score 0–3 on: requirements, estimates, workload separation, boundary reasoning, failure analysis, and communication. A 3 requires naming a rejected alternative and the measurement that would trigger it later.

