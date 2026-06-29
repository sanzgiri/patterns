# Microservices Architecture

**Aliases:** —
**Category:** Architectural style
**Sources:**
[microservices.io](https://microservices.io/patterns/microservices.html) ·
[Neo Kim](https://systemdesign.one/system-design-interview-cheatsheet/) ·
[ByteByteGo](https://github.com/ByteByteGoHq/system-design-101) ·
Sam Newman — *Building Microservices* (2nd ed., 2021) ·
Martin Fowler & James Lewis — [bliki/Microservices](https://martinfowler.com/articles/microservices.html)

---

## Problem

> [!TIP]
> **ELI5.** Your one shared house with three hundred housemates is now unbearable. Every renovation needs everyone's approval; the kitchen schedule is a nightmare; one person's bad cooking ruins dinner for all. Time for each family to get their own apartment.

The monolith problems described in [monolith.md](monolith.md) compound past a certain scale: every team blocks every other team on releases, the codebase is too big for any one person to hold in their head, a single hot module forces the entire app to be over-provisioned, a memory leak in one feature crashes the whole product. At that point the system needs to be split — but how, and along what lines?

The naive split — "one service per CRUD entity" — produces the worst of both worlds: distributed-systems complexity *plus* monolith-style tight coupling. The pattern's real problem is therefore not "how to split" but "how to split *along the right seams* so each piece is genuinely independent."

## How it works

> [!TIP]
> **ELI5.** Many small programs instead of one big one, each with its own database. They talk to each other over the network — usually by REST/gRPC for requests and by events for notifications. Each program is owned, deployed, and scaled by one team independently.

A microservices architecture decomposes the application into **small, independently deployable services**, each:

1. Aligned to a **single business capability** (orders, billing, search) — not a layer (DAO, controller) and not a technical role (cache, queue).
2. Owning its **own data store** — no other service reads or writes those tables.
3. Communicating with other services only over a **network boundary** — synchronous request/response or asynchronous events.
4. Owned by a **single team** that builds, deploys, and operates it independently.

![Microservices structure](../diagrams/svg/microservices-structure.svg)

A request from the **Client** enters through the **API Gateway** (orange), which routes by URL prefix or host to the correct service: `/orders/*` to the **Orders Service**, `/users/*` to the **Users Service**. Each service is its own runtime process (often its own container, often its own programming language) and owns its **own database**: Orders DB, Inventory DB, Billing DB, Users DB. Crucially, no service reads another service's database directly — that would re-introduce the monolith's coupling at the schema level, which is the worst kind because schemas are invisible contracts.

When services need to coordinate, they do so in one of two ways. **Synchronously** for queries that need an immediate answer ("does this user exist?"), via REST or gRPC. **Asynchronously** for notifications and downstream effects, via an **Event Bus** like Kafka, RabbitMQ, or NATS. In the diagram, when the Orders Service creates an order, it emits an `OrderPlaced` event; the Inventory Service consumes that event to decrement stock, and the Billing Service consumes the same event to create an invoice — neither knowing the other exists. This is the central trick: events let services react without being coupled to the producer's code or release cycle.

The other half of the pattern is the **deployment story**:

![Microservices deployment](../diagrams/svg/microservices-deployment.svg)

Each service has its own **repository**, its own **CI/CD pipeline**, and its own **runtime footprint** — and most importantly, ships on its own schedule. Orders can deploy v2.3 four times today; Inventory may not ship a release this week. Each service can run more or fewer instances based on its own load (Orders 4 pods, Inventory 2, Billing 3). Teams choose their own languages, frameworks, and databases — within sensible org-wide guardrails (a "golden path" template, observability standards, security baseline).

This independence is the whole point — but the *cost* is real. Now you need: service discovery, an API gateway, a message broker, distributed tracing, centralized logging, contract testing between services, schema-evolution discipline on events, saga or eventual-consistency thinking for any operation that spans more than one service, and dramatically more operational sophistication. Most teams that adopt microservices and then regret it underestimated this cost.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Modular Monolith** | Same module boundaries, no network. The right *first* step almost always. |
| **Self-Contained Services** | Coarser than microservices; each owns its UI too. Less granular, less overhead. |
| **Macroservices** | Pragmatic middle ground — a few large services rather than dozens of small ones. |
| **Service-Oriented Architecture (SOA)** | Predecessor. Heavier per-service (XML, WS-*, ESB-mediated). |
| **Service Mesh** | The infrastructure layer (Envoy sidecars + control plane) that makes microservices traffic management, mTLS, and observability tractable. |
| **Database per Service** | The data-side rule that distinguishes microservices from "distributed monolith." |

## When NOT to use

- **Small teams or small products** — see [monolith.md](monolith.md). Below ~10 engineers, microservices almost always cost more than they save.
- **Domain boundaries you don't yet understand.** Premature service boundaries calcify; refactoring across a network boundary is *much* harder than across a function call.
- **Strong cross-feature transactional requirements everywhere.** If most business operations span 3+ services, sagas become the dominant cost; reconsider the boundaries.
- **No appetite for the operational investment** — service mesh, observability, contract testing, on-call rotations per service. These are non-negotiable.

---

## Real-world implementations (infrastructure / frameworks)

| Tool | Role |
|---|---|
| **Kubernetes** | The de-facto deployment substrate; provides service discovery, health checks, scaling, rollouts. |
| **Istio / Linkerd / Consul Connect** | Service meshes — sidecars + control plane for mTLS, traffic shifting, circuit breaking. |
| **Envoy** | The data-plane proxy underneath most service meshes. |
| **Kafka / Pulsar / RabbitMQ / NATS** | Event buses for async inter-service communication. |
| **gRPC / OpenAPI** | Standard service-to-service RPC and contract description. |
| **OpenTelemetry / Jaeger / Tempo** | Distributed tracing. |
| **Pact / Spring Cloud Contract** | Consumer-driven contract testing. |

## Companies using it (notable examples)

| Company | Use | Status |
|---|---|---|
| **Netflix** | The case study that popularized microservices at internet scale. Originated Hystrix, Eureka, Zuul, Ribbon — most of which inspired the cloud-native ecosystem. | ✅ Verified — [Netflix Tech Blog (multi-post series, 2012–2018)](https://netflixtechblog.com/) |
| **Amazon** | Famous internal mandate (Bezos memo, 2002) to expose all functionality as services; widely cited origin of the practice at scale. | ✅ Verified — Bezos memo well-documented in many secondary sources; corroborated in [*Working Backwards*, Bryar & Carr, 2021] |
| **Uber** | Grew to 1000+ services; has since consolidated some back ("microservices to macroservices"). | ✅ Verified — [Uber Engineering, *Microservice Architecture at Uber*](https://www.uber.com/blog/microservice-architecture/) |
| **Spotify** | Squad/tribe model with hundreds of services; their org chart was the template for "team-aligned services". | ⚠ Well-documented in talks and blog posts; specific link not re-fetched |
| **Airbnb** | Migrated from a Rails monolith to a service-oriented + microservices architecture. | ⚠ Multiple engineering blog posts; not re-verified |
| **Twitter** | Decomposed the Rails monolith to JVM services around 2010–2014 (Finagle, Mesos, Aurora — early service mesh / scheduler stack). | ⚠ Discussed extensively in talks (Marius Eriksen et al.); not re-verified |

**⚠ marks claims widely known industry-wide but not re-verified by primary-source fetch.**

---

## Further reading

- Sam Newman, *Building Microservices*, 2nd ed. (2021) — the standard textbook.
- Sam Newman, *Monolith to Microservices* (2019) — the migration companion.
- Martin Fowler & James Lewis, *Microservices* (2014) — [the original article](https://martinfowler.com/articles/microservices.html).
- Chris Richardson, *Microservices Patterns* (2018) — pattern-language treatment; companion to microservices.io.
- Werner Vogels (Amazon CTO), *Modern applications at AWS* — talks on service-oriented architecture at hyperscale.

---

*Diagram sources: [`../diagrams/src/microservices-structure.d2`](../diagrams/src/microservices-structure.d2), [`../diagrams/src/microservices-deployment.d2`](../diagrams/src/microservices-deployment.d2).*
