# Strangler Fig

**Aliases:** Strangler Application, Strangler Pattern, Incremental Migration via Facade
**Category:** Migration / Architectural evolution
**Sources:**
[Microsoft Azure Architecture Center — Strangler Fig pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig) ·
[Martin Fowler, *StranglerFigApplication* (2004)](https://martinfowler.com/bliki/StranglerFigApplication.html) ·
[Chris Richardson — microservices.io: Strangler Application](https://microservices.io/patterns/refactoring/strangler-application.html)

---

## Problem

> [!TIP]
> **ELI5.** You have a giant old system that does a hundred things. You want to replace it with modern services — but you can't shut it down for a year while you rebuild. The trick: put a smart router in front, and migrate features **one at a time**. Each migrated feature goes to a new service; everything else still hits the legacy. Slowly, the new services strangle the old one — until eventually the legacy has nothing left to do, and you switch it off.

A "big bang" rewrite of a critical legacy system almost always fails. The classic war stories — Netscape's rewrite, the Chrysler C3 project, Borland's repeated rewrites — show the same pattern: the team spends months/years building the replacement, the business keeps adding requirements to the old system in parallel, the gap never closes, the replacement is abandoned. Meanwhile, the legacy is still in production, accruing complexity.

But you can't just leave the legacy alone forever. Old systems get harder to modify, depend on dying technology, and become single points of failure. You need to migrate — without freezing development and without betting the business on a multi-year rewrite.

Martin Fowler named the **Strangler Fig** pattern after the rainforest plant that grows around a host tree, gradually taking it over until the host dies and the strangler stands alone. The pattern: build the new system around the old one, route progressively more traffic to it, and eventually decommission the old system.

## How it works

> [!TIP]
> **ELI5.** Put a facade (a router or reverse proxy) in front of the legacy. Initially everything is routed to legacy. Build one new microservice for one piece of functionality. Update the facade to route requests for that piece to the new service. Repeat for each piece. When all functionality is migrated, the legacy is unused — turn it off.

The pattern unfolds in phases:

![Strangler Fig phases](../diagrams/svg/strangler-fig-phases.svg)

**Phase 1: Insert a facade.** Put a reverse proxy, API gateway, or routing layer in front of the legacy. The facade is initially a transparent pass-through — every request still goes to the legacy. This step alone provides nothing functional; its only purpose is to give you a *routing point* you can change later without touching clients.

**Phase 2: Extract the first capability.** Pick the easiest, lowest-risk capability to migrate first (typically "low blast radius, high learning value" — e.g., user profile reads, not order processing). Build it as a new microservice. Update the facade's routing rules: "Requests matching `/users/profile/*` go to the new service; everything else still goes to legacy."

**Phase 3: Extract progressively more.** Each migration is a small, reversible change at the facade. The team builds new services in parallel with old features running. Routing rules accumulate. The new services grow; the legacy shrinks (in usage, not yet in code).

**Phase 4: Decommission the legacy.** When all functionality has been migrated, the legacy receives no traffic. You can turn it off, archive it, or eventually remove the facade too (if it's no longer needed).

The pattern's key insight is that **the facade decouples migration from client changes**. Clients don't know which backend serves them; the facade can switch any time. This lets the team migrate at a controlled pace, validate each migration in production, and roll back instantly if needed.

### The facade as a control plane

The facade is more than a router — it's a control plane for the migration itself:

![Strangler Fig: routing rules at the facade](../diagrams/svg/strangler-fig-routing.svg)

Routing rules are evaluated in priority order, with the legacy as the catch-all default. Each rule can dispatch differently:

- **Hard cutover**: 100% of matching traffic goes to the new service.
- **Canary**: route a small percentage (5%, 25%, 50%) to the new service; fall back to legacy on error.
- **Shadow traffic**: send the request to *both* — return the legacy response to the client, and asynchronously compare what the new service produced. Catches behavioral drift before customers see it.
- **Feature-flag gated**: route by user, organization, region, or any attribute. Used heavily for trusted-beta rollouts.
- **Dark launch**: send all traffic to both, return legacy's response, validate the new service's response shape and performance at production load *before* depending on it.

These strategies are what makes the strangler safe. A well-instrumented facade gives the team confidence that each migration is correct *before* committing to it.

### Database strangling

The trickiest part is the database. If the legacy and new services share a database, you haven't actually decoupled — the new service is still constrained by the legacy schema, and the legacy can corrupt data the new service wrote. The hard version of the strangler pattern requires **database migration as well**:

1. **Co-existence phase**: new service writes to its own database AND writes through to legacy (or vice-versa). Use Change Data Capture (CDC), dual-write, or [Outbox](https://microservices.io/patterns/data/transactional-outbox.html) to keep them in sync.
2. **Reconciliation**: continuously compare both stores; alert on drift; eliminate sources of drift.
3. **Cut writes**: stop writing to legacy DB; legacy reads from new DB (or via API).
4. **Cut reads**: legacy no longer needs the data; new service is source of truth.

This is where most strangler migrations *actually* live for the longest time. The technology routing is usually weeks; the data sync and cutover is months.

### When the legacy can't be touched

Sometimes the legacy is so brittle that even adding a facade in front of it is risky. The pattern can be adapted: use **CDC** to mirror legacy writes into a new event stream, build new services against the stream, and incrementally swap the UI from legacy-rendered pages to new-service-rendered pages. The "facade" becomes an event-bus + a new UI layer rather than a request-routing proxy.

### Common failure modes

- **Strangler in name only**: team renames the project "strangler" but still does a big-bang rewrite of half the system in one PR. The pattern requires real *incrementality*.
- **Facade becomes a god service**: too much logic in the facade itself (auth, transformation, orchestration), turning it into a new monolith.
- **No retirement plan**: the legacy lingers for years because no one prioritizes decommissioning. Without an explicit retirement deadline, strangling stalls.
- **Database left behind**: technology routing migrates; data never does. The team ends up with two systems both depending on the legacy DB.
- **No clear rollback**: a buggy new service can't be rolled back because the facade was updated but the legacy was also modified. Keep the legacy frozen during migration.

### Compared to other approaches

- **Big-bang rewrite**: high-risk; usually fails. Strangler trades duration for safety.
- **Branch by abstraction** (Trunk-Based Development pattern): used *inside* a monolith for code-level refactoring. Strangler is the system-level analog.
- **Encapsulate Legacy** (Joshi/microservices.io): a related pattern where the legacy is wrapped in a service interface but not yet replaced — often a prerequisite to strangling.
- **Side-by-Side Replacement**: dual-run two complete systems; doesn't decouple components, just runs old and new in parallel.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Classic Strangler (Fowler)** | HTTP facade + incremental capability extraction. |
| **Event-driven Strangler** | CDC-fed event stream; new services react to legacy data without HTTP. |
| **Branch by abstraction** | Code-level variant inside a single codebase. |
| **Parallel run / Side-by-side** | Run both systems on every request; compare results. Often a phase within a strangler. |
| **Encapsulate Legacy** | Wrap legacy in a service interface (often a prerequisite). |
| **Anti-Corruption Layer** | Pair with strangler: the new service uses an ACL to call the still-present legacy. |
| **Database Strangler** | Same pattern applied to data storage (often the harder half). |

## When NOT to use

- **System is small enough to rewrite quickly** — a 3-month rewrite may beat a 12-month strangler.
- **No mid-stream routing point possible** — desktop app with no central gateway is hard to strangle.
- **Legacy will be retired wholesale anyway** — e.g., vendor sunset with hard deadline; just plan the cutover.
- **No team capacity to maintain two systems in parallel** — strangler requires running both for a long time.
- **Legacy too brittle to instrument** — sometimes a rewrite is genuinely faster than understanding the legacy's bugs.

---

## Real-world implementations

| Tooling layer | What you use |
|---|---|
| **Facade / router** | NGINX, HAProxy, AWS ALB rules, Cloudflare Workers, Envoy, Kong, AWS API Gateway, Apigee |
| **Service mesh as facade** | Istio / Linkerd routing rules |
| **CDC for data strangler** | Debezium, AWS DMS, Maxwell, Striim |
| **Feature flags for gated routing** | LaunchDarkly, Split.io, Unleash, ConfigCat |
| **Shadow traffic tooling** | GoReplay, Diffy (Twitter), AWS Traffic Mirroring, Envoy's mirror filter |
| **Process orchestration** | Camunda, Temporal — sometimes used as the migration brain |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Amazon** | The classic 2000s migration from the Obidos monolith to services used strangler-style migration. | ✅ Verified — Werner Vogels talks, [microservices.io case studies](https://microservices.io/articles/scopes.html) |
| **Netflix** | Migrated from a monolithic DVD-rental system to streaming services using strangler-style refactoring. | ✅ Verified — Netflix Tech Blog |
| **Shopify** | Public engineering blog series on incrementally extracting services from the Shopify monolith ("the modular monolith → strangler hybrid"). | ✅ Verified — [Shopify Engineering blog](https://shopify.engineering/) |
| **Monzo Bank** | "From monolith to microservices" — explicit strangler pattern. | ✅ Verified — Monzo blog & QCon talks |
| **GitHub** | Migrated the Rails monolith piece-by-piece, with Trilogy and other extractions. | ✅ Verified — GitHub Engineering blog |
| **Stack Overflow / Stack Exchange** | Have publicly discussed staying on the "modular monolith" *instead* of strangling — counterexample worth knowing. | ✅ Verified — Nick Craver's blog |

---

## Further reading

- Martin Fowler, *StranglerFigApplication* (2004) — the foundational essay. [martinfowler.com/bliki/StranglerFigApplication.html](https://martinfowler.com/bliki/StranglerFigApplication.html).
- Microsoft Azure Architecture Center, *Strangler Fig pattern* — practical guidance with Azure-specific tooling.
- Chris Richardson, *microservices.io* — Strangler Application pattern entry.
- Sam Newman, *Monolith to Microservices* (O'Reilly, 2019) — entire book devoted to strangler-style migration; required reading.
- Sam Newman, *Building Microservices* (O'Reilly, 2nd ed. 2021) — Ch on migration.
- Adam Tornhill, *Your Code as a Crime Scene* — analyzing legacy code to plan strangler migrations.
- Michael Feathers, *Working Effectively with Legacy Code* — code-level techniques that complement system-level strangling.

---

*Diagram sources: [`../diagrams/src/strangler-fig-phases.d2`](../diagrams/src/strangler-fig-phases.d2), [`../diagrams/src/strangler-fig-routing.d2`](../diagrams/src/strangler-fig-routing.d2).*
