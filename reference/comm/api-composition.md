# API Composition

**Aliases:** Read-Time Aggregation, Fan-out / Fan-in Query, In-Memory Join, Aggregator Microservice
**Category:** Communication / Query
**Sources:**
[Chris Richardson — microservices.io: API Composition](https://microservices.io/patterns/data/api-composition.html) ·
Sam Newman, *Building Microservices* (2nd ed.), Ch on querying patterns ·
Discussion in BFF (Sam Newman 2015) and CQRS literature

---

## Problem

> [!TIP]
> **ELI5.** A client asks "show me the customer profile with their recent orders and loyalty status." With [database-per-service](../data/database-per-service.md), no single database holds all that data. You can't JOIN across service databases. Solution: write a small piece of code that calls each service in parallel and combines the results in memory. That code is the **API Composer**.

In a microservices architecture, [each service owns its own database](../data/database-per-service.md). This independence is essential but creates a real problem: **queries that span multiple services**.

A client (web app, mobile app, partner API) often wants data that lives across services:

- "Show customer Alice's profile with her recent 5 orders and current loyalty tier" → spans Customers + Orders + Loyalty.
- "Show this product with its reviews, current stock, and applicable promotions" → spans Catalog + Reviews + Inventory + Pricing.
- "Show the user's dashboard with messages, notifications, and feed" → spans Messaging + Notifications + Activity Feed.

With a shared database, this is trivial — one SQL JOIN. With per-service databases, there's no SQL JOIN across them; you must fetch from each service and combine.

You have two main strategies:

- **API Composition** (this pattern): fetch from each service at query time, combine in memory.
- **[CQRS read model](../data/cqrs.md)**: pre-build a denormalized view fed by events; query the view.

API composition is simpler and gives fresher data; CQRS gives faster reads but is more infrastructure. The choice depends on latency, freshness, and complexity requirements.

## How it works

> [!TIP]
> **ELI5.** Write a composer (a function, a BFF endpoint, or a dedicated service) that:
> 1. Receives the client's request.
> 2. Calls multiple services in parallel.
> 3. Waits for all responses.
> 4. Combines the results into one response.
> 5. Returns it.
> The composer hides the multiple-service mess from the client.

The structure:

![API Composition: aggregate from multiple services](../diagrams/svg/api-composition.svg)

The composer can live in several places:

- **An API Gateway** doing aggregation (Kong, Apigee, AWS API Gateway with Lambda integrations).
- **A [BFF](../comm/bff.md)** — backend-for-frontend that aggregates for a specific client.
- **A dedicated aggregator microservice** — its only job is composition.
- **The client itself** (in some architectures) — but typically not preferred because of multi-service coupling on clients.

The composer's logic:

```python
async def get_customer_summary(customer_id):
    # Parallel fan-out
    customer, orders, loyalty = await asyncio.gather(
        customers_client.get(customer_id),
        orders_client.list(customer_id=customer_id, limit=5),
        loyalty_client.get(customer_id),
    )

    return {
        "customer": customer,
        "orders": orders,
        "loyalty": loyalty,
    }
```

The fan-out is **parallel**: latency is the *max* of the service latencies, not the sum. If each service takes 50ms, total response time is ~50ms (plus a small composition overhead), not 150ms.

### The hard parts

API composition looks trivial; the complications are operational:

**1. Partial failure.** What if one service is down? Options:
- **All-or-nothing**: if any service fails, return 500. Simplest, often wrong.
- **Graceful degradation**: return what's available; mark missing parts as `null` or omit them. Often the right call.
- **Required vs optional**: distinguish — if Customers is down, return 503 (can't render anything useful); if Loyalty is down, return data without loyalty tier.

A typical composer has timeout, circuit-breaker, and degradation rules per downstream.

**2. Over-fetching.** The client wanted only `customer.name` and `last_order.total`. The composer fetched complete customer and complete orders. Wasted bandwidth and CPU on both sides. Mitigations:
- **Field selection**: composer accepts a list of fields to include and asks each service for only those.
- **GraphQL** (see [GraphQL Federation](graphql-federation.md)) — the client expresses exact field needs; framework handles it.
- **Per-use-case BFF endpoints**: define a specific endpoint for each client need; fetch exactly what's needed.

**3. N+1 problem.** "Enrich each order with the product name." Naive code: 1 call for orders + N calls for products (one per order). Bad.
- **Batch endpoint**: products service exposes `/products?ids=1,2,3,...` for bulk lookup.
- **DataLoader pattern** (Facebook): collect requests within a tick, deduplicate, batch.
- **Pre-projection**: if this is common, build a [CQRS read model](../data/cqrs.md) that already joins.

**4. Caching.** Composer can cache per-service results independently. Customer profile might be cached for 5 minutes; orders for 30 seconds; loyalty status for an hour. Each cache TTL reflects how often that data changes.

**5. Consistency.** Reads from three services aren't guaranteed to be consistent. The composer fetches customer at T1 and orders at T2 — they might reflect different snapshots. For most read paths this is fine; for cases where it matters, you need a different approach (transactional consistency requires same-DB queries or careful version-pinning).

### When API composition is the right choice

API composition shines when:

- **Simple aggregation** — composing existing data shapes without complex transformations.
- **Freshness matters** — you want the latest data, not eventually-consistent snapshots.
- **Modest query patterns** — a handful of well-known query shapes, not arbitrary client queries.
- **Low-to-medium volume** — fan-out scales O(services × QPS); at extreme scale, becomes expensive.

It's the wrong choice when:

- **Complex queries** (search, full-text, aggregations across services, filtering by remote attributes) — those need a [CQRS read model](../data/cqrs.md) with pre-built indexes.
- **Very high read volume** — every query fans out; consider read model.
- **Strict consistency needed** across services — fan-out can't give it.
- **Clients need arbitrary query flexibility** — consider [GraphQL Federation](graphql-federation.md).

### Compared to CQRS read models

Both solve "I need data from multiple services in one query," but trade differently:

![API Composition vs CQRS read model](../diagrams/svg/composition-vs-cqrs.svg)

**API composition** does the join at *read time*. Always fresh; simple to start; doesn't need event streams. Pays latency on every query.

**CQRS read model** does the join at *write time* (via events). Very fast reads; complex queries supported; pays infrastructure cost for the read model itself.

Most mature systems use **both**: composition for simple, fresh aggregations; read models for complex queries (search, dashboards, reports). The BFF or composer chooses which strategy per query.

### Patterns within composition

A few sub-patterns appear repeatedly:

- **Fan-out by ID list**: client provides IDs; composer calls service X for each (with batching).
- **Sequential composition**: must call A to get data needed for B's call. Slower than parallel; sometimes unavoidable.
- **Conditional composition**: only call service Y if A's result indicates it's relevant.
- **Hedged composition**: send to two replicas of the same service; use whichever responds first.
- **Per-field composition** (GraphQL-like): compose at the field level, not the resource level.

### Where composition runs

The composer can live in:

- **API Gateway** with aggregation features (Kong, Apigee, AWS API Gateway with Lambda).
- **BFF** for client-specific aggregation (see [BFF](bff.md)).
- **Dedicated composer service** for shared aggregations.
- **GraphQL gateway** as a specialized composer (see [GraphQL Federation](graphql-federation.md)).
- **Edge function** (Cloudflare Workers, Vercel Functions) for low-latency client-side aggregation.

The choice depends on who needs the aggregation. Client-specific: BFF. Shared across clients: dedicated composer or gateway. Type-safe and flexible: GraphQL.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **API Composer Service** | Dedicated service whose only job is aggregation. |
| **API Gateway aggregation** | Gateway with multi-backend aggregation feature. |
| **BFF aggregation** | Per-client BFF that composes for its client. |
| **GraphQL Federation** | Type-safe, field-level composition. |
| **CQRS Read Model** | Pre-built denormalized view; alternative for complex queries. |
| **In-process composition** | Same-process aggregation within a monolithic service. |
| **Edge composition** | Aggregation at CDN/edge functions; low client-side latency. |

## When NOT to use

- **Complex queries** with cross-service filtering, sorting, aggregation — use CQRS read model.
- **Very high read volume** — composition fan-out becomes expensive.
- **Strict consistency needed** — composition is best-effort.
- **Many subqueries with high latency** — sum of latencies dominates; consider redesign.

---

## Real-world implementations

| Tool / Approach | Notes |
|---|---|
| **Custom Node/Go/Python BFF** | The most common — just write a service. |
| **AWS API Gateway with Lambda** | Gateway invokes a Lambda that composes multiple downstream calls. |
| **Kong / Apigee** | Have built-in aggregation plugins. |
| **Apollo GraphQL Server** | Composition via GraphQL resolvers. |
| **Apollo Federation Gateway** | Field-level composition across subgraphs. |
| **GraphQL Mesh / Hasura** | Hybrid composition over GraphQL, REST, gRPC, OpenAPI. |
| **Cloudflare Workers** | Edge composition for low-latency client paths. |
| **NestJS / Spring Cloud Gateway** | Framework-level support for composition. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Netflix** | BFFs and composer services aggregate from microservice backends. | ✅ Verified — Netflix Tech Blog series on BFF / device APIs |
| **Airbnb** | BFFs and composers for web and mobile experiences. | ✅ Verified — Airbnb Engineering blog |
| **Shopify** | API composition + GraphQL for both Storefront API and Admin API. | ✅ Verified — Shopify dev documentation |
| **Spotify** | Multiple composer-style services for client experiences. | ✅ Verified — Spotify Engineering blog |
| **GitHub** | GraphQL API for newer integrations is essentially a composer. | ✅ Verified — GitHub GraphQL API docs |
| **Twitch, Pinterest** | Composer patterns in BFFs. | ⚠ Mentioned in conference talks |
| **Most large microservices orgs** | API composition is universal at the BFF / gateway layer. | ✅ Common pattern |

---

## Further reading

- Chris Richardson, *microservices.io* — API Composition pattern.
- Sam Newman, *Building Microservices* (2nd ed.) — Ch on querying patterns.
- Newman, *Backends For Frontends* (essay) — composition in BFFs.
- Apollo GraphQL documentation — composition via resolvers and federation.
- *Designing Data-Intensive Applications* (Kleppmann), Ch 11 — analytical alternatives.
- Multiple Netflix Tech Blog posts on composer / aggregator services.
- *Building Microservices with Spring*, various authors — concrete Spring-based composition.

---

*Diagram sources: [`../diagrams/src/api-composition.d2`](../diagrams/src/api-composition.d2), [`../diagrams/src/composition-vs-cqrs.d2`](../diagrams/src/composition-vs-cqrs.d2).*
