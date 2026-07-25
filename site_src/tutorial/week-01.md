# Week 1 — Requirements, Boundaries, and Scaling Shape

!!! abstract "Learning goal"
    Turn an ambiguous product request into explicit requirements, estimate its scale, identify the first bottleneck, and choose a sensible first architecture.

## Lesson: design from forces

Do not begin with “Should we use microservices?” Begin with the forces acting on the system:

- users and actions;
- read/write volume and data growth;
- latency and availability;
- consistency and durability;
- privacy, security, and regional constraints;
- failure and recovery expectations.

Product requirements describe what users want. System guarantees describe what the architecture must preserve.

> “Users can upload photos” is a product requirement. “An acknowledged upload is not lost after a regional failure” is a system guarantee.

### Workload separation

A photo-sharing service has at least two very different workloads:

- **Control plane:** metadata, permissions, sharing, users, and transactions.
- **Data plane:** large binary objects, thumbnails, downloads, and delivery bandwidth.

Keeping these separate lets the application tier handle metadata while object storage and a CDN handle large bytes.

### Monolith versus services

A monolith is often the right starting point when domain boundaries, transaction boundaries, and scaling requirements are still being discovered. Split a component when it has a distinct scaling shape, reliability boundary, data ownership, or team ownership.

Microservices introduce independent deployment and scaling, but also network calls, distributed debugging, versioning, and distributed data consistency.

## Worked example: global photo sharing

Assume:

- 100 million daily active users;
- 10 uploads per user per month;
- 5 MB average original photo;
- reads are 20× more frequent than writes;
- global users.

Roughly:

```text
1 billion photos/month
≈ 385 uploads/second average
≈ 4,000 uploads/second at a 10× peak
≈ 5 PB/month of original image storage
```

The first obvious bottleneck is likely image delivery bandwidth and origin load, not the metadata database.

```text
Client ── metadata ──> API ──> metadata database
   │
   └── direct upload/download ──> object storage ──> CDN ──> viewers

object-created event ──> thumbnail worker
                     └──> moderation worker
                     └──> search indexer
```

The original photo must be durable before upload acknowledgement. Thumbnails, moderation results, search indexing, and public-feed propagation can usually be asynchronous.

## Practice: now answer

1. What five requirements would you clarify before finalizing this design?
2. Which guarantee should be strongest: upload durability, thumbnail freshness, or feed freshness? Why?
3. A celebrity’s photo receives 30 million requests in one minute. What fails first?
4. What would cause you to split media processing or search into separate services?

## Interview drill

Design a photo-sharing service in 35 minutes. Use this order:

1. Clarify requirements.
2. Estimate scale.
3. Draw the simplest architecture.
4. Deep-dive into the first bottleneck.
5. Walk through one failure.
6. Explain one rejected alternative.

!!! success "Self-check"
    A strong answer states assumptions, separates control-plane metadata from data-plane bytes, uses asynchronous processing for derived work, and explains why the first design does not need microservices everywhere.
