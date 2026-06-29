# Event Sourcing

**Aliases:** Event Sourcing, ES, Append-Only Event Log, Event-Sourced Aggregate, Event Store
**Category:** Architecture / Data
**Sources:**
[Martin Fowler — *Event Sourcing* (2005)](https://martinfowler.com/eaaDev/EventSourcing.html) — the canonical article ·
[Greg Young — *CQRS, Task Based UIs, Event Sourcing agh!*](https://goodenoughsoftware.net/2012/03/02/cqrs-task-based-uis-event-sourcing-agh/) ·
[microservices.io — Event Sourcing](https://microservices.io/patterns/data/event-sourcing.html) ·
[Microsoft Azure — Event Sourcing pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) ·
*Domain-Driven Design* (Eric Evans, 2003) — original "domain events" idea ·
*Designing Data-Intensive Applications* (Kleppmann, 2017) — Ch 11 on event logs ·
[EventStoreDB documentation](https://www.eventstore.com/) ·
[Vaughn Vernon — *Implementing Domain-Driven Design* (2013)](https://kalele.io/books/) — practical DDD + ES

---

## Problem

> [!TIP]
> **ELI5.** In a normal database, when you change something, the old value is gone. The row `account=42, balance=1500` doesn't tell you HOW the balance got to 1500 — was it one deposit, ten? Was anything refunded? When was each change? That history is **gone**, overwritten by the last UPDATE. **Event sourcing** flips this: instead of storing current state, store every change as an immutable **event** (`AccountOpened`, `MoneyDeposited`, `MoneyWithdrawn`). Current state is computed by **replaying** events. The event log IS the database. You get a complete history for free, plus the ability to add new "views" (projections) at any time by replaying — a fundamentally different model from CRUD.

The motivating insight: **the history of what happened is more fundamental than the snapshot of current state**. In real-world domains:

- A bank account isn't a balance; it's a sequence of deposits and withdrawals. The balance is *derived*.
- A customer order isn't a status; it's a sequence of placed/paid/picked/shipped/delivered/returned events. The current status is *derived*.
- A shopping cart isn't a list of items; it's items added, removed, quantity changed. The current cart is *derived*.
- A document isn't its content; it's a sequence of edits. The current content is *derived*.

CRUD databases preserve the current snapshot but discard the history. That works for systems where history doesn't matter (configuration, lookup tables) but is fundamentally lossy for systems where it does (financial, audit, collaboration, regulated industries).

**Event sourcing** says: keep the events. Compute state from them. The implications are profound:

- Complete audit trail — automatic.
- Time travel — "what was the state on date X?"
- New views — add a new way of looking at the data; replay events to populate it.
- Bug fixes — found a bug in derived state? Fix the projection code, replay events, the state is correct.
- Domain modeling — events naturally express what happened in business terms ("OrderPlaced," not "row inserted into orders table").
- Integration — events become the natural integration point for downstream systems (CDC for free).

The trade-offs are equally profound: querying current state requires replay (or maintained projections); schema evolution of events is hard; not all domains benefit. Event sourcing is *not* the default — it's the right answer for some domains (finance, audit-heavy systems, complex domain logic, event-driven architectures) and overkill for others (simple CRUD apps).

The pattern was named by Fowler in 2005 but its roots are older — financial ledgers, version control systems (git's commits are events), and the actor model all share the DNA. Greg Young's talks and writing in 2010-2015 popularized the modern form, often paired with CQRS ([page](../data/cqrs.md)).

## How it works

> [!TIP]
> **ELI5.** Every change becomes an event written to an append-only log. To get current state, replay all events for that entity (or use a snapshot to skip ahead). To make queries fast, maintain **projections**: derived "current state" views that get updated as new events arrive. New projection? Replay the event log from the beginning to populate it. Buggy projection? Delete it, fix the code, replay to rebuild.

### Core mechanics

The pattern at its simplest:

![Event sourcing concept](../diagrams/svg/event-sourcing.svg)

Instead of:
```sql
UPDATE account SET balance = 1500 WHERE id = 42;
```

You write:
```
APPEND TO Account-42 EventLog: MoneyDeposited { amount: 200, source: 'ach', time: ... }
```

Current state is computed by folding over all events:

```python
def current_balance(account_id):
    balance = 0
    for event in event_log.read(stream=f"Account-{account_id}"):
        if isinstance(event, AccountOpened):
            balance = 0
        elif isinstance(event, MoneyDeposited):
            balance += event.amount
        elif isinstance(event, MoneyWithdrawn):
            balance -= event.amount
    return balance
```

For an account with 5 events, this is fast. For an account with 5 million events, it's slow — solved by **snapshots** (more below).

### Events: the data model

Events have specific properties:

- **Immutable**: once written, never modified or deleted. Period.
- **Past tense**: the name describes something that happened. `OrderPlaced`, not `PlaceOrder` (that's a command).
- **Business-meaningful**: named in the language of the domain. `PaymentDeclined`, not `RowInsertedIntoTransactionsTable`.
- **Self-contained**: includes all information needed to interpret it (data + context).
- **Versioned**: as the system evolves, event schemas change; versioning is critical.
- **Ordered**: typically a sequence number / position within a stream.
- **Stamped**: timestamp, correlation ID, causation ID, actor.

Typical structure:

```json
{
  "event_id": "evt-7f3a-...",
  "event_type": "MoneyDeposited",
  "event_version": 2,
  "stream_id": "Account-42",
  "sequence": 47,
  "timestamp": "2024-01-15T10:23:45Z",
  "data": {
    "amount": 200,
    "currency": "USD",
    "source": "ach",
    "external_ref": "ach-tx-..."
  },
  "metadata": {
    "actor": "user-alice",
    "correlation_id": "req-...",
    "causation_id": "evt-..."   // the command/event that caused this
  }
}
```

### Commands vs events

A subtle but critical distinction:

- **Command**: intent. "Place an order." May be rejected (validation, business rules). Imperative tense.
- **Event**: fact. "Order placed." Cannot be rejected — it already happened. Past tense.

The flow in event-sourced systems:
1. Command arrives (`PlaceOrder { customer, items }`).
2. Command handler loads the aggregate (replays events).
3. Command handler validates the command against current state.
4. If valid, command handler produces one or more events (`OrderPlaced`, `InventoryReserved`).
5. Events are atomically appended to the log.
6. Events are processed by projections (read-side updates) and other subscribers.

The command/event distinction maps onto CQRS: commands write events; queries read projections.

### CQRS + Event Sourcing: natural marriage

Event sourcing and [CQRS](../data/cqrs.md) are commonly paired — though they're independently useful:

![Event sourcing + CQRS architecture](../diagrams/svg/event-sourcing-cqrs.svg)

The architecture:

**Write side (commands)**:
1. Command handler loads aggregate from event log.
2. Validates against current state.
3. Appends new events to log atomically.

**Event bus / log**:
- Kafka, EventStoreDB, Postgres append-only table, custom event store.
- Events published as they're appended.

**Read side (projections)**:
- Multiple projection handlers subscribe to events.
- Each maintains its own optimized view: current balance in KV, monthly statements in SQL, fraud dashboard in Elasticsearch, ML features in a feature store.
- Each can be rebuilt by replaying events from start.

**Queries**: hit projections, not the event log. Fast, indexed, optimized per use case.

This is the architectural sweet spot for event sourcing: write side is simple and consistent (just append events); read side is fast and flexible (specialized projections); evolution is graceful (add projections by replay; never touch the source of truth).

### Snapshots: making replay tractable

For aggregates with millions of events, replaying every event for each query is too slow. Snapshots cut the cost: periodically save the current state, and start replay from the snapshot.

```
state = read_snapshot(stream)              # state at event N
events = read_events(stream, after=N)      # events after N
current = fold(events, initial=state)
```

Snapshot strategy:
- **Every N events** (e.g., every 100): simple.
- **On schedule** (e.g., nightly): time-based.
- **Adaptive**: based on stream length.

Snapshots are derived data — they're rebuildable from events, so they don't need their own durability guarantees beyond performance.

### Schema evolution: the hard part

Events are immutable, but the *meaning* of events evolves over time. Code from 2026 must still correctly interpret events written in 2020. This is the hardest practical aspect of event sourcing.

Strategies:

- **Versioned events**: each event has a version field; readers handle multiple versions.
- **Upcasters**: code that transforms old-version events to new-version on read.
- **Weak schema**: events are JSON; tolerate missing/extra fields.
- **Strict schema with registry**: Avro/Protobuf with schema registry; strict compatibility rules.
- **Copy-and-transform migrations**: rare and expensive — copy events to a new stream while transforming.

The discipline: **never break old events**. Add new fields with defaults; never remove fields. New events can use new fields. Reader code handles all historical versions.

This is similar to managing a long-term wire protocol. Greg Young's *Versioning in an Event Sourced System* (free PDF) is the practical reference.

### Eventual consistency between write and read

Because projections are updated asynchronously, there's a window where the event has been written but the projection hasn't caught up. Implications:

- After issuing a command, the user might not immediately see their change reflected in queries.
- Two queries milliseconds apart might see different state.
- "Read your own writes" requires special care (read from write side, or wait for projection catch-up, or include event version in URL).

Mitigations:
- **Synchronous projection updates** for critical views (couples write and read; defeats the purpose somewhat).
- **Versioned reads** ("show me projection ≥ event N"), letting clients wait.
- **UI optimism**: assume the change will succeed; show the new state immediately.
- **Pull mode**: clients can subscribe to projection updates.

Most event-sourced systems accept eventual consistency on the read side — it's a fundamental trade-off.

### When to use event sourcing

**Strong fit**:
- **Financial / accounting**: ledgers are events by nature; audit requirements demand it.
- **Audit-heavy / regulated**: SOX, HIPAA, healthcare — automatic audit trail is a huge win.
- **Complex domain logic** that changes over time — events let you re-derive state when business rules evolve.
- **Multiple views needed** — many projections beat many CRUD tables.
- **Time-travel queries** — "what was the state on date X?"
- **Collaborative editing** (operational transforms, CRDTs build on event sourcing).
- **Event-driven microservices** — events as integration unit.
- **Long-running business processes** with many state transitions.

**Bad fit**:
- **Simple CRUD apps** — overkill; UPDATE is fine.
- **No domain complexity** — just data in, data out.
- **No need for history** — config tables, lookup data.
- **Team without DDD experience** — event sourcing rewards (and demands) domain modeling discipline.
- **Tight latency** without snapshots/projections planned — naive replay is slow.

### Anti-patterns

- **Storing current state in the events** ("BalanceSetTo: 1500") — defeats the point; events should describe what *changed*, not the resulting state.
- **CRUD events** (`AccountUpdated`) — events should be business-meaningful, not generic.
- **Mutable events** — any mutation breaks the model.
- **Skipping snapshots for long streams** — replay becomes prohibitive.
- **Synchronous projections** for every read — couples write/read; defeats CQRS benefits.
- **Coarse-grained events** that bundle too much — hard to evolve; hard to project differently.
- **Fine-grained events** that fragment what should be atomic — multiple writes for one logical change.
- **Treating events as deletable/editable** — when a "delete" is needed, write a "compensating event."
- **Ignoring versioning until production** — every event written before versioning is a compatibility burden.

### Replay performance

Naive replay of all events for one entity gets slow with stream length. Practical numbers:

- 10K events: fine (milliseconds).
- 100K events: needs snapshots.
- 1M+ events: needs snapshots + parallelism.

For multi-tenant systems, partitioning event log by entity (stream-per-entity, not global stream) is essential for replay scalability.

### Storage backends

| Backend | Notes |
|---|---|
| **EventStoreDB** | Purpose-built event store; rich subscription model |
| **Kafka** | Most common in practice; treated as event log + processing |
| **AxonServer** | Java/Axon framework backend |
| **Marten (Postgres)** | Postgres-backed; .NET/Java |
| **Apache Pulsar** | Event log + topics |
| **EventStore / Kurrent** | Specialized event store |
| **DIY on Postgres / DynamoDB** | Append-only table with sequence; common starting point |
| **AWS DynamoDB Streams + S3** | Cloud-native option |
| **Apache Iceberg / Delta Lake / Hudi** | For big-data ES |

### Tooling and frameworks

| Framework | Language | Notes |
|---|---|---|
| **Axon Framework** | Java/Kotlin | Comprehensive ES + CQRS |
| **EventStoreDB SDKs** | Polyglot | Native client for EventStoreDB |
| **NEventStore, Marten, EventFlow** | .NET | .NET ES libraries |
| **Eventide** | Ruby | Ruby ES toolkit |
| **Akka Persistence / Lagom** | Scala/Java | Actor-model ES |
| **Cadence / Temporal** | Polyglot | Workflow + ES |
| **Apache Flink, Kafka Streams** | Polyglot | Stream processing for projections |
| **Slick, EventuateDB** | Polyglot | Newer entrants |

### Trade-offs

Advantages:
- **Complete history**: full audit, time travel, debug-by-replay.
- **Flexible projections**: add new views by replay.
- **Natural integration**: events are the integration medium.
- **Domain modeling**: events express business meaning.
- **Bug-by-replay debug**: fix code, replay; state recovers.
- **Strong fit with DDD, CQRS, event-driven**.
- **Decouples write and read** scaling.

Disadvantages:
- **Eventual consistency** read-side.
- **Complex querying** — current state via projections.
- **Schema evolution** of events is the hard part.
- **Snapshot management** for performance.
- **Storage growth** linear with history.
- **Mental model shift** for team.
- **Tooling investment** — most teams need to build ES infrastructure.
- **Overkill for simple domains**.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Event Sourcing (pure)** | Events as source of truth. |
| **CQRS** ([page](../data/cqrs.md)) | Separate write/read models; often paired with ES. |
| **Event-driven architecture** | Events as integration medium; not necessarily sourced. |
| **Domain events (DDD)** | Events expressing domain transitions; subset of ES. |
| **Outbox** ([page](../data/outbox.md)) | Lighter pattern: state + outbox emits events. |
| **Change Data Capture (CDC)** ([page](../data/cdc.md)) | Capture changes from existing DB; CRUD-friendly alternative. |
| **Snapshots** | Performance optimization. |
| **Saga / process manager** ([page](../data/saga.md)) | Long-running coordination on events. |
| **Operational Transform / CRDT** ([page](../data/crdt.md)) | Collaborative editing on event log. |
| **Audit log** ([page](../ops/audit-log.md)) | Specialization for compliance. |
| **Append-only log** (Kafka, replicated log) | Storage primitive ES builds on. |

## When NOT to use

- **Simple CRUD apps** without rich domain.
- **No regulatory or audit need**.
- **Team without DDD experience and discipline**.
- **Tight latency on reads** without projection investment.
- **Heavy ad-hoc OLAP queries** against the event store directly — use a warehouse for that.
- **As a magic bullet** — event sourcing doesn't fix bad models; it forces them to be explicit.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **EventStoreDB / Kurrent** | Purpose-built event store |
| **Kafka** | Event log + processing |
| **Axon Framework + AxonServer** | Java ES+CQRS |
| **Marten** | Postgres-backed (.NET) |
| **Eventide** | Ruby |
| **Akka Persistence** | Scala/Java actors |
| **Apache Pulsar** | Event log |
| **Apache Flink, Kafka Streams** | Stream processing for projections |
| **Cadence / Temporal** | Workflow + ES |
| **NEventStore, EventFlow** | .NET libraries |
| **EventuateDB** | Distributed ES |
| **AWS EventBridge + DynamoDB Streams** | Cloud-native |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Walmart Labs** | Heavy event sourcing in fulfillment systems. | ✅ Verified — Walmart engineering blog |
| **ING Bank** | Event-sourced core banking systems. | ✅ Verified — ING engineering blog |
| **Banks generally** | Ledger systems are event-sourced by nature. | ✅ Industry standard |
| **Booking.com** | Reservation systems with event-sourced concepts. | ⚠ Implied; specifics vary |
| **Uber** | Cadence (event-sourced workflow engine, OSS). | ✅ Verified — Uber Engineering blog |
| **Netflix** | Internal event sourcing for various domain services. | ⚠ Mentioned in talks |
| **Microsoft (Azure)** | Azure Event Grid + various internal event-sourced services. | ✅ Verified — Azure docs |
| **GitHub** | Git itself is event-sourced; some internal systems too. | ✅ Verified (git) |
| **Most modern fintech** | Plaid, Stripe, Square — heavily event/ledger based. | ⚠ Architecture implied |
| **Most CQRS adopters** | Often paired with ES. | ✅ Common combination |

---

## Further reading

- Martin Fowler — *Event Sourcing* (2005).
- Greg Young — *CQRS Documents* (free PDF).
- Greg Young — *Versioning in an Event Sourced System* (free book) — the practical guide to event schema evolution.
- *Implementing Domain-Driven Design* (Vaughn Vernon, 2013) — DDD + ES + CQRS.
- *Versioning in an Event-Sourced System* (Young) — schema evolution.
- *Designing Data-Intensive Applications* (Kleppmann, 2017) — Ch 11.
- microservices.io — Event Sourcing, CQRS, Saga patterns.
- Microsoft Azure Architecture Center — Event Sourcing pattern.
- EventStoreDB documentation and Greg Young's talks.
- Axon Framework documentation — practical Java ES.
- *Patterns, Principles, and Practices of Domain-Driven Design* (Millett & Tune).

---

*Diagram sources: [`../diagrams/src/event-sourcing.d2`](../diagrams/src/event-sourcing.d2), [`../diagrams/src/event-sourcing-cqrs.d2`](../diagrams/src/event-sourcing-cqrs.d2).*
