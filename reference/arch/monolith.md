# Monolithic Architecture

**Aliases:** Monolith, Single-tier app, Big-ball-of-mud (pejorative variant)
**Category:** Architectural style
**Sources:**
[microservices.io](https://microservices.io/patterns/monolithic.html) ·
[Neo Kim](https://systemdesign.one/system-design-interview-cheatsheet/) ·
[ByteByteGo](https://github.com/ByteByteGoHq/system-design-101) ·
Sam Newman — *Monolith to Microservices* (2019)

---

## Problem

> [!TIP]
> **ELI5.** You and your friends share one big house. Cooking, sleeping, working — everything happens under one roof. Easy when there are three of you. Becomes very awkward when there are three hundred and you all need to renovate at the same time.

For a small team and a small app, building everything as one deployable unit is *correct*. The problem only emerges as the system grows along three axes:

1. **Codebase complexity** — every change risks touching every other feature; tests run forever; one developer's bug blocks the whole release train.
2. **Deployment coupling** — you can't ship the billing fix without also shipping the (untested) change to inventory.
3. **Scaling waste** — the whole app must be replicated together even when only one module (search, image processing) is hot.

The "problem" of the monolith pattern is therefore really *recognizing when you've outgrown it*, not the pattern itself. Many teams either jump out of it too early (and pay distributed-systems tax for no reason) or stay in it too long (and grind to a halt).

## How it works

> [!TIP]
> **ELI5.** One program, one process, one deploy. All the modules live inside it and call each other with normal function calls. One database underneath. Simple, fast, and exactly right until it isn't.

A monolithic application packages **all functional modules into a single deployable artifact** — a JAR, a container image, a binary — and runs them in a single process. Modules communicate by in-process function calls, share an address space, and typically share one database.

![Monolith structure](../diagrams/svg/monolith-structure.svg)

In the structure above, the **Application Process** is the whole product. A request from the **Client** hits the **Load Balancer**, lands on the process's Web/API layer, and from there flows through whatever internal modules are involved — Orders calls Inventory and Billing as ordinary library calls, with no network in between. Every module reads and writes the **Shared Database** directly. There is exactly one schema, one connection pool, one transaction boundary, one deployable.

This is *not* the same as "badly designed." A well-modularized monolith — sometimes called a **modular monolith** — keeps the modules in the diagram as distinct, well-bounded packages with clear interfaces, and is often the right architecture for years. The boundaries are enforced at compile time rather than at the network, which is both cheaper and *more* reliable than the equivalent microservice setup.

Scaling a monolith is straightforward and almost embarrassingly effective for a long time:

![Monolith scaling by replication](../diagrams/svg/monolith-scaling.svg)

You run **N identical copies** of the binary behind a load balancer. Each instance is stateless (sessions in Redis or signed cookies, files in object storage), so any request can hit any instance. The instances share the single database. You scale *horizontally* by adding instances — which works until the database itself becomes the bottleneck, at which point you reach for read replicas, then partitioning, then a serious conversation about decomposing the monolith.

The monolith's enduring strengths are the same ones it has always had: **one place to look** when debugging, **transactional consistency** across modules via the shared DB, **trivial deployment**, **no network failures between modules**, and **fast end-to-end development** because there are no cross-service contracts to negotiate. The weaknesses become real when team size or code size makes those properties impossible to preserve: a deploy now requires coordinating dozens of teams; a slow query in one module starves another; one OOM crashes the entire product.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Modular Monolith** | A monolith with strict internal module boundaries enforced at the package/build level. Often the right intermediate step before extracting microservices — and often the right *final* architecture too. |
| **Self-Contained Service** | A monolith-per-business-capability; each owns its UI, logic, and data. A pragmatic middle ground between monolith and microservices. |
| **Microservices Architecture** | The opposite end of the spectrum (see [microservices.md](microservices.md)). |
| **Strangler Fig** | The migration path *from* a monolith *to* something else. |

## When NOT to use

- **Many independent teams that need to ship at different cadences.** Monolith's coupled releases will become the dominant cost.
- **Workloads with vastly different scaling profiles per module** (one tiny CPU-bound module, one enormous GPU workload). Replicating the whole app to scale one slice is wasteful.
- **Hard fault-isolation requirements.** A monolith shares a process; one bad allocation crashes everything.

## When to *prefer* it (despite the prevailing fashion)

- **Small teams (≤ 10 engineers).** Microservices' overhead almost always dominates the benefit at this size.
- **Early-stage products where the bounded contexts are still being discovered.** Premature service boundaries are extremely expensive to undo.
- **Internal tools, admin apps, batch jobs.** The reasons microservices exist (independent scaling, polyglot teams) usually don't apply.

---

## Real-world implementations

There is nothing to install — the monolith is the *default*. Tooling commonly cited as **monolith-friendly**: Ruby on Rails, Django, Laravel, Spring Boot (single-app mode), ASP.NET Core, Phoenix. The "modular monolith" school has standard patterns: Java's JPMS modules, .NET's project boundaries, Maven multi-module builds, Cargo workspaces.

## Companies using it (notable examples)

| Company | Use | Status |
|---|---|---|
| **Shopify** | Famously runs one of the largest Rails monoliths in production; has invested heavily in modular-monolith tooling rather than splitting. | ✅ Verified — [*Deconstructing the Monolith*, Shopify Engineering, 2019](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity) |
| **Basecamp / HEY** | Explicitly defends "majestic monolith" as their architecture. | ✅ Verified — [DHH, *The Majestic Monolith*, 2016](https://m.signalvnoise.com/the-majestic-monolith/) |
| **Stack Overflow** | Operates a monolith on a small number of large servers; well-documented architecture. | ✅ Verified — [Nick Craver, *Stack Overflow Architecture series*](https://nickcraver.com/blog/2016/02/03/stack-overflow-a-technical-deconstruction/) |
| **GitHub** | Core product is a large Rails monolith; has gradually extracted services around it but the center remains monolithic. | ⚠ Well-known via GitHub Engineering blog posts; specific link not re-verified here |
| **Etsy** | PHP monolith for many years; has decomposed gradually. | ⚠ Discussed in conference talks; not re-verified |

**⚠ marks claims widely known industry-wide but not re-verified by primary-source fetch.**

---

## Further reading

- Sam Newman, *Monolith to Microservices* (2019) — the standard reference for understanding when, why, and how to decompose.
- DHH (David Heinemeier Hansson), *The Majestic Monolith* and *The Citadel* — strong arguments for staying monolithic.
- Shopify Engineering, *Deconstructing the Monolith* — large-scale modular-monolith experience report.
- Martin Fowler, *MonolithFirst* — [martinfowler.com/bliki/MonolithFirst.html](https://martinfowler.com/bliki/MonolithFirst.html).

---

*Diagram sources: [`../diagrams/src/monolith-structure.d2`](../diagrams/src/monolith-structure.d2), [`../diagrams/src/monolith-scaling.d2`](../diagrams/src/monolith-scaling.d2).*
