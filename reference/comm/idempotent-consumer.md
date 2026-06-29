# Idempotent Consumer

**Aliases:** Idempotent Receiver, Message Deduplication, Effectively-Once Processing
**Category:** Async messaging / Resilience
**Sources:**
[Chris Richardson — microservices.io: Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html) ·
[Microsoft Azure Architecture Center — Idempotent message processing](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/interservice-communication) ·
Discussion in Kafka, RabbitMQ, SQS documentation

---

## Problem

> [!TIP]
> **ELI5.** Almost every message broker delivers messages **at-least-once** — meaning duplicates are normal, not exceptional. A consumer that crashes after processing but before acknowledging will receive the same message again. A producer that retries after a timeout will publish duplicates. If your consumer naively does "send notification" or "charge card" on every message, duplicates cause real damage. The fix: make the consumer **idempotent** — design it so processing the same message twice has the same effect as processing it once.

In distributed messaging systems, **at-most-once** delivery (no duplicates) is unachievable in the general case. Brokers therefore default to **at-least-once** semantics — they will redeliver a message if they're not sure the consumer processed it. Common causes of duplication:

- **Consumer crash after processing but before ack.** Broker doesn't know the message was handled; redelivers.
- **Network blip during ack.** Broker thinks ack failed; redelivers.
- **Producer retries on timeout.** Producer sent, didn't get an ack in time, retries; broker now has two copies.
- **Rebalancing in Kafka consumer groups.** Partition reassignment can cause overlap.
- **Manual replay** from operations (rebuilding a service, backfilling a downstream).

In every realistic deployment, **duplicates happen**. A consumer that processes "charge customer $100" twice is a lawsuit. A consumer that sends two welcome emails is annoying. A consumer that decrements inventory twice causes overselling.

The defense isn't "try harder to make duplicates not happen" — that's a doomed quest. The defense is **idempotency**: design the consumer so processing the same message N times has the same observable effect as processing it once.

## How it works

> [!TIP]
> **ELI5.** Before processing a message, check whether you've already processed one with the same ID. If yes, just acknowledge and move on (no work done). If no, process it, record that you've processed it, then acknowledge. The "record" must be in the same database transaction as the side-effects, so a crash between them can't leave you partially done.

The core pattern:

![Idempotent Consumer: at-least-once delivery + dedup](../diagrams/svg/idempotent-consumer.svg)

```python
def on_message(msg):
    if already_processed(msg.id):
        ack()                        # skip silently
        return

    with db.transaction():
        process(msg)                 # business logic; side effects
        db.execute("INSERT INTO processed_messages (id) VALUES (%s)", msg.id)
    ack()
```

Every message has a stable **idempotency key** — typically a UUID assigned by the producer. The consumer keeps a **dedup store** recording which IDs it has already processed. Before processing, check; if seen, skip. After processing (and recording), ack.

The critical correctness property: **the side effects and the dedup record must be atomic**. If you process the message (charging the card) and then crash before recording the dedup, the next delivery will charge again. The atomicity is what makes this safe.

### Where the idempotency key comes from

The key needs to be assigned by the producer (or derived from the message content) and propagate unchanged through any retries:

- **Producer-assigned UUID**: producer creates a UUID per message; retries use the same UUID. Cleanest.
- **Natural key**: some messages have a natural unique key (e.g., "order id 42 placed" — the order_id + event_type is unique).
- **Hash of content**: hash the full payload to derive a key. Works if message content is deterministic.
- **Kafka offset (sometimes)**: in Kafka, the (topic, partition, offset) tuple is unique per delivery; can serve as a dedup key. Limited usefulness because rebalancing can shift offsets.

The Outbox pattern naturally provides this: the outbox row's UUID becomes the message ID and the dedup key downstream.

### Where the dedup store lives

The dedup record needs to be in a transactional store that's atomic with the side-effects:

- **Same database as the side-effect**: ideal. The same transaction that processes the message also writes the dedup record. Atomicity is guaranteed by the DB.
- **Per-message-ID column with UNIQUE constraint**: the simplest implementation. Try to INSERT; if it fails with unique-violation, the message was already processed. No separate check needed.
- **Redis SET with TTL**: lightweight, suitable when business logic is external (e.g., calls a third-party API). Less atomicity guarantee — if Redis says "new" but the side-effect fails, the message stays unprocessed but appears processed.
- **Kafka Streams transactional / exactly-once support**: Kafka has its own transactional semantics for cross-topic effects, but only within Kafka.

The **same DB + UNIQUE constraint** approach is usually the right choice. It piggybacks on the database's existing atomicity; no additional infrastructure needed.

### Idempotency at different layers

Idempotency can be implemented at multiple layers:

- **Application layer** (the pattern above): the consumer checks a dedup store.
- **Domain layer**: the business operation is naturally idempotent. E.g., `SET inventory = 100` is idempotent; `DECREMENT inventory BY 10` is not. Where possible, prefer naturally-idempotent operations.
- **Database layer**: use UNIQUE constraints, UPSERT, or `INSERT ... ON CONFLICT DO NOTHING` to make the write itself idempotent.
- **Infrastructure layer**: some systems support producer-side dedup (Kafka's idempotent producer eliminates duplicates *from the same producer instance*; doesn't cover cross-producer or cross-process retries).

The right answer often combines layers. A consumer might have application-level dedup *and* use UPSERT for the DB write, providing defense-in-depth.

### Effectively-once vs Exactly-once

"Exactly-once" delivery is impossible in distributed systems in the general case (the FLP result and related). "**Effectively-once**" processing — at-least-once delivery plus idempotent consumer — gives the same end-state behavior. This is the standard model for production messaging.

Kafka's "exactly-once semantics (EOS)" works within Kafka's own boundary — read-process-write where both source and sink are Kafka topics. It uses Kafka's transactional protocol. It doesn't help when the side-effect is external (charging a card, sending an email) — for those, you still need application-level idempotency.

### Pitfalls

- **No idempotency key**: messages without stable IDs make dedup impossible. Producers must assign and preserve IDs.
- **Dedup store and side-effect not atomic**: crash between them causes either silent duplication or silent loss.
- **Dedup TTL too short**: if you only remember messages for 1 hour, a 2-hour-delayed redelivery slips through.
- **Dedup store performance**: at high throughput, the dedup lookup itself can be a bottleneck. Index appropriately; consider sharding.
- **Cross-consumer dedup needed?**: usually each consumer dedupes its own processing. If multiple consumers should *together* process once, you need shared dedup state — more complex.
- **Natural idempotency assumed but not present**: "DELETE WHERE id = X" is idempotent (deleting an already-deleted row is fine), but "DELETE WHERE customer = X AND most_recent" is not. Audit carefully.
- **Side-effects to external systems**: external systems may not let you check "did I already do this?" — you need to send your idempotency key in headers (Stripe's `Idempotency-Key`, AWS's `X-Amzn-Trace-Id`-style).

### When you really can't be idempotent

A few side-effects are genuinely non-idempotent — sending a one-time push notification, ringing a phone. For these:

- **Accept the small risk** of occasional duplicate. Most cases (push notification heard twice) are acceptable.
- **Apply additional dedup at the destination** (the recipient's device dedupes by notification ID).
- **Use producer-side dedup that's stronger than at-least-once**: Kafka's idempotent producer at minimum; transactional broker support where available.

The principle: never assume "won't be duplicated"; design for at-least-once and handle the consequence.

### The discipline

Idempotency is a *discipline* across producers and consumers:

- Producers: assign stable IDs; preserve them on retry.
- Brokers: propagate IDs faithfully; don't transform them.
- Consumers: check dedup before processing; record dedup atomically with side-effects.
- Operators: monitor dedup-store growth; clean up old entries; alert on missing IDs.

When the discipline is applied consistently, the entire system behaves effectively-once — duplicates exist but are silently handled, side-effects happen once.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Dedup table with UNIQUE constraint** | Simplest; relies on DB to reject duplicate INSERT. |
| **Dedup check + insert in transaction** | Check first; insert after side-effect. |
| **Redis SET with TTL** | External dedup store; less atomic. |
| **UPSERT (`INSERT ... ON CONFLICT`)** | Database-layer idempotency via UPSERT semantics. |
| **Natural idempotency** | Design operations to be naturally idempotent (`SET x = 5` vs `INCREMENT x`). |
| **Kafka EOS (Exactly-Once Semantics)** | Within Kafka boundary; not for external side-effects. |
| **Stripe-style Idempotency-Key header** | Pass key to external service that respects it. |
| **Effectively-once processing** | The combination of at-least-once delivery + idempotent consumer. |

## When NOT to use

- **Side-effects are not visible** (e.g., metric increments where double-count is acceptable).
- **At-most-once delivery is acceptable** (rare; mostly fire-and-forget telemetry).
- **Producer can guarantee no duplicates** in a closed system.
- **Operation is naturally idempotent** (UPSERT, SET) and dedup table is unnecessary overhead.

---

## Real-world implementations

| Tool / Pattern | How |
|---|---|
| **Database UNIQUE constraint on message ID** | Simplest production-grade dedup. |
| **Spring Integration / Spring Cloud Stream** | Built-in idempotent receiver support. |
| **Stripe `Idempotency-Key` header** | Stripe API respects keys for 24 hours; replays return original response. |
| **AWS SQS FIFO dedup** | Per-queue 5-minute dedup window. |
| **Kafka Streams EOS** | Transactional read-process-write within Kafka. |
| **Debezium downstream consumers** | Standard pattern in CDC pipelines. |
| **DBOS / Temporal / Inngest** | Durable execution frameworks that subsume idempotency. |
| **Eventuate Tram** | Java library for messaging with built-in idempotency. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Stripe** | The `Idempotency-Key` header is the most-documented production idempotency pattern. | ✅ Verified — [Stripe API docs](https://stripe.com/docs/api/idempotent_requests) |
| **AWS** | SQS FIFO queues, Lambda processing, many services use idempotency keys. | ✅ Verified — AWS docs |
| **Shopify** | Public engineering posts on idempotent webhook processing. | ✅ Verified — Shopify Engineering blog |
| **DoorDash, Wix, Uber** | Multiple engineering posts on idempotent event consumers. | ✅ Verified — respective engineering blogs |
| **Confluent / Kafka users** | Kafka EOS is the standard for intra-Kafka effectively-once. | ✅ Verified — Confluent docs |
| **Every payment system** | Idempotency is mandatory; PCI / regulatory requirement. | ✅ Universal |

---

## Further reading

- Stripe's blog post *Designing robust and predictable APIs with idempotency* — the most-cited production write-up. [stripe.com/blog/idempotency](https://stripe.com/blog/idempotency).
- Chris Richardson, *microservices.io* — Idempotent Consumer pattern.
- Microsoft Azure Architecture Center — Idempotent message processing.
- *Enterprise Integration Patterns* (Hohpe & Woolf, 2003) — the foundational catalog; "Idempotent Receiver" is a chapter.
- Confluent blog — Kafka EOS and consumer-side idempotency.
- Newman, *Building Microservices* (2nd ed.) — Ch on event-driven communication and idempotency.
- *Designing Data-Intensive Applications* (Kleppmann), Ch 11 — covers exactly-once vs effectively-once.

---

*Diagram sources: [`../diagrams/src/idempotent-consumer.d2`](../diagrams/src/idempotent-consumer.d2).*
