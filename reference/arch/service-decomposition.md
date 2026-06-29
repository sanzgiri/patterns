# Service Decomposition (by Business Capability / by Subdomain)

**Aliases:** Decompose by Business Capability, Decompose by Subdomain, Bounded Context Decomposition, DDD-Driven Decomposition
**Category:** Architectural decomposition
**Sources:**
[Chris Richardson — microservices.io: Decompose by Business Capability](https://microservices.io/patterns/decomposition/decompose-by-business-capability.html) ·
[Chris Richardson — microservices.io: Decompose by Subdomain](https://microservices.io/patterns/decomposition/decompose-by-subdomain.html) ·
Eric Evans, *Domain-Driven Design* (2003) ·
Vaughn Vernon, *Implementing Domain-Driven Design* (2013)

---

## Problem

> [!TIP]
> **ELI5.** You've decided to break a monolith into microservices. But **into what?** Where do the boundaries go? The wrong cut creates "microservices" that are really a distributed monolith — services that always change together and depend on each other for every request. The right cut creates services that evolve independently, owned by independent teams. Two complementary approaches help find the right boundaries: **business capability** (what the business does) and **subdomain** (what the business knows about). They usually agree; when they don't, you have to make a judgment call.

The single hardest question in microservices is: **where do the service boundaries go?** Get this wrong and you get the worst of both worlds — the complexity of distributed systems with none of the agility benefits.

Bad decompositions are common:

- **By technical layer**: a "web team" owns frontends across all features, an "API team" owns endpoints, a "business logic team" owns rules, a "database team" owns schemas. Every feature change requires coordination across all four teams. Conway's Law disaster.
- **By entity**: every domain entity becomes a service (UserService, OrderService, ProductService, AddressService). Services are too fine-grained, every operation requires N service calls, transactions become impossible.
- **Arbitrarily**: based on engineer availability ("Bob will own the orders microservice") or initial guesses that aren't revisited.

These decompositions miss the point. The reason microservices help is **enabling independent teams to evolve independently**. Service boundaries should be drawn where independence is genuine — where one team can change one service without coordinating with others.

Two complementary heuristics from microservices.io and from DDD help find good boundaries: **business capability** and **subdomain**.

## How it works

> [!TIP]
> **ELI5.** Two ways to find service boundaries. **Business capability**: ask "what does our company do?" — each major thing the business does (order management, payment processing, shipping) becomes a service. **Subdomain (DDD)**: ask "what distinct concepts does our domain have?" — places where the same word means different things, or where different teams of experts speak different languages, are natural boundaries. They usually agree. When they don't, choose what aligns better with how your teams actually work.

### Decompose by Business Capability

The simpler heuristic: look at the org chart and the **major capabilities the business provides**:

![Decompose by Business Capability vs Subdomain](../diagrams/svg/decomposition-approaches.svg)

A retail company might identify:
- Order Management
- Inventory Management
- Customer Management
- Payment Processing
- Shipping & Fulfillment
- Pricing & Promotions
- Recommendations

Each becomes a service (or a small group of related services for a large capability). The boundaries often align with departments or teams that already exist — and that's good. Conway's Law says system structure mirrors org structure; deliberately aligning them avoids fighting it.

The strength of this approach: it's **easy to start with**. You can list business capabilities in a workshop in an hour. The boundaries are obvious to business stakeholders. Communication with non-engineers is natural.

The weakness: business capabilities can be coarse. "Order Management" might be one service or fifteen, depending on what's inside.

### Decompose by Subdomain (DDD)

The DDD heuristic, from Eric Evans: identify the distinct **subdomains** of your business and make each one a **bounded context** — a service with its own model, ubiquitous language, and team.

DDD classifies subdomains:
- **Core subdomain**: where you compete. Amazon's recommendations and pricing engine. The investment goes here.
- **Supporting subdomain**: necessary but not differentiating. Customer account management. Build but don't over-invest.
- **Generic subdomain**: commodity, often buy or rent. Authentication, email sending, payments-via-Stripe. Don't build it yourself unless you have unusual needs.

Each subdomain has a **ubiquitous language** — terms with specific meanings within that context. "Order" in the Sales context might be a draft cart; in the Fulfillment context it's a confirmed thing to ship; in the Billing context it's a financial document. These are *different concepts*, even with the same name. The Subdomain boundary draws the line.

The strength: rooted in deep understanding of the business. Boundaries reflect real conceptual seams. Highly resistant to drift over time.

The weakness: requires substantial domain modeling effort upfront. Hard to do without an experienced DDD practitioner.

### Where they agree (usually)

In practice, **the two approaches usually converge on the same boundaries**:

| Business capability | Subdomain (DDD) |
|---|---|
| Order Management | Sales / Order subdomain |
| Inventory Management | Inventory subdomain |
| Customer Management | Customer / Identity subdomain |
| Payment Processing | Payments subdomain |
| Shipping & Fulfillment | Fulfillment subdomain |

This convergence isn't coincidence — both approaches are reaching for the same thing: **areas of the business that have their own internal logic, vocabulary, and stakeholders**. The DDD approach gets there via concepts; the capability approach gets there via activities. Same answer, different routes.

### When they disagree

Sometimes they don't agree. Common cases:

- A business capability spans multiple subdomains. "Customer service" might include identity, profiles, preferences, and communications — distinct concepts that DDD would split.
- A subdomain crosses business capabilities. "Pricing rules" might apply to both Orders and Subscriptions — a single subdomain, but two capabilities.

When they disagree, lean on practical considerations:

- **Team structure**: who's going to own it? A boundary that splits team work is friction.
- **Rate of change**: areas that change together stay together. If A and B always change in lockstep, splitting them just creates coordination.
- **Data ownership**: clear data boundaries usually indicate clear service boundaries.
- **Domain expert opinion**: ask the people who understand the business.

### The hardest part: avoid technical-layer decomposition

The most-common mistake — far more common than getting capabilities or subdomains wrong — is **decomposing by technical layer**:

![A worked decomposition example](../diagrams/svg/decomposition-example.svg)

Bad: a frontend team owns all frontends; an API team owns all APIs; a business-logic team owns rules; a database team owns schemas. Every feature requires coordination across all four. Lead time is months.

Good: each service team owns a vertical slice end-to-end. Frontend, API, business logic, database — all owned by the same team. The team can ship a feature without coordinating with anyone else.

This vertical-slice principle is what makes microservices actually pay off. If your "microservices" are still owned by horizontal teams, you've changed nothing fundamental.

### Sizing services

A common question: **how big should a service be?** No fixed answer, but useful heuristics:

- **Two-pizza team**: small enough for one team (~6–10 engineers) to own.
- **Independently deployable**: a service should ship without coordination with others.
- **Cohesive purpose**: a service should answer a single "what does this do?" question.
- **Owning its own data**: a service should fully own a meaningful piece of state.

If a service is bigger than that, consider splitting. If it's smaller (nano-services), consider merging — the operational cost of many tiny services often dominates.

### Iteration is normal

Decomposition isn't done once. As you learn about the business and watch the services in production, you'll see:

- Services that always change together → consider merging.
- Services where one team owns multiple but they're internally split by accident → consider re-grouping.
- Services where ownership has shifted teams → consider splitting along the new team line.

Mature microservices organizations periodically revisit their decomposition. Service boundaries are not architectural decisions made once; they're ongoing engineering work that requires attention.

### Common pitfalls

- **Decomposing by technical layer.** The big one — kills all benefit.
- **Decomposing by entity.** "OrderService", "LineItemService", "AddressService" — too fine, every operation becomes N calls.
- **Premature decomposition.** Splitting before understanding the domain. Start with a modular monolith; split when you understand.
- **Not considering data ownership.** Services that share databases aren't really independent.
- **Ignoring team structure.** A service that no team naturally owns becomes nobody's problem.
- **Not iterating.** Original boundaries become stale; failure to revisit.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Decompose by Business Capability** | Looking at what the business does. |
| **Decompose by Subdomain** | DDD-based; looking at distinct concepts. |
| **Bounded Context (DDD)** | The DDD term for a subdomain-aligned service. |
| **Event Storming** | Workshop technique to discover boundaries via business events. |
| **Strategic Design (DDD)** | The DDD ceremony for finding bounded contexts. |
| **Team Topologies stream-aligned teams** | Org-aligned approach; teams own slices end-to-end. |
| **Modular Monolith** | First step: modular boundaries inside one deploy, evolve to services later. |
| **Self-Contained Systems (SCS)** | Variant emphasizing per-service UI as well as data. |
| **Database-per-Service** | Pair with — gives services real independence. |

## When NOT to use

- **Small team / small system** — a well-structured monolith is simpler.
- **Without domain understanding** — premature decomposition based on guesses.
- **For purely technical concerns** (caching, sharding, etc.) — those aren't services.
- **Where business doesn't naturally split** — some domains genuinely have one tightly-coupled core.

---

## Real-world implementations (techniques / workshops)

| Technique | Use |
|---|---|
| **Event Storming** | Workshop with domain experts; sticky notes for events, commands, aggregates. Reveals boundaries. |
| **Domain Storytelling** | Narrating user stories with pictograms; surfaces actors and their interactions. |
| **Context Mapping** | DDD strategic exercise; maps relationships between bounded contexts. |
| **Capability Mapping (TOGAF / Business Architecture)** | Enterprise-architecture technique. |
| **Microservices Mapping (microservices.io)** | Chris Richardson's structured exercise. |
| **Team Topologies workshops** | Skelton & Pais's framework for team + system co-design. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Amazon** | The "two-pizza team" principle drove service decomposition; teams own services end-to-end. | ✅ Verified — Werner Vogels talks |
| **Netflix** | Public engineering posts describe DDD-influenced service boundaries. | ✅ Verified — Netflix Tech Blog |
| **Shopify** | Modular-monolith approach with planned future decomposition by subdomain. | ✅ Verified — Shopify Engineering blog |
| **Monzo** | Public talks on how they decomposed (~1,500 microservices, business-capability aligned). | ✅ Verified — QCon talks |
| **Spotify** | Squad/Tribe model is essentially Team Topologies; each squad owns its services. | ✅ Verified — Spotify Engineering posts |
| **DDD community case studies** | Many published examples from major insurers, banks, retailers. | ✅ Verified — DDD Europe, Explore DDD talks |

---

## Further reading

- Chris Richardson, *microservices.io* — both decomposition patterns and their relationships.
- Eric Evans, *Domain-Driven Design* (2003) — the foundational DDD text. Especially Part IV (Strategic Design).
- Vaughn Vernon, *Implementing Domain-Driven Design* (2013) — more concrete than Evans; better for engineers.
- Vlad Khononov, *Learning Domain-Driven Design* (O'Reilly, 2021) — modern, accessible DDD.
- Skelton & Pais, *Team Topologies* — org-design lens on service decomposition.
- Sam Newman, *Building Microservices* (2nd ed.) — Ch on decomposition.
- Alberto Brandolini, *Introducing EventStorming* — the EventStorming method.

---

*Diagram sources: [`../diagrams/src/decomposition-approaches.d2`](../diagrams/src/decomposition-approaches.d2), [`../diagrams/src/decomposition-example.d2`](../diagrams/src/decomposition-example.d2).*
