# Anti-Corruption Layer (ACL)

**Aliases:** ACL, Translator Layer, Adapter Layer (DDD)
**Category:** Integration / Migration
**Sources:**
[Microsoft Azure Architecture Center — Anti-Corruption Layer](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer) ·
[Eric Evans, *Domain-Driven Design* (2003), Ch 14 — Bounded Context patterns](https://www.domainlanguage.com/ddd/) ·
[Chris Richardson — microservices.io: Anti-corruption layer](https://microservices.io/patterns/refactoring/anti-corruption-layer.html)

---

## Problem

> [!TIP]
> **ELI5.** You're building a new service that needs to talk to an old system whose data model is messy — weird field names, magic strings, fields that mean different things in different contexts, conventions nobody remembers why. If you let those weird concepts leak into your new code, your new service becomes just as messy. The fix: build a translator layer that hides the legacy mess; your code only sees clean concepts.

When a new service must integrate with a legacy system — or a third-party API, or another team's service with a very different domain model — the legacy's accidental complexity threatens to **infect** the new code. Concrete examples of "infection":

- The new `Order` class has a `legacy_status_code` field because the legacy uses cryptic two-character codes (`'01'`, `'02'`, `'9'`) and "we'll translate later."
- The new service uses the legacy's customer-ID format (because that's what the legacy returns), even though it has nothing to do with the new domain.
- Date fields are passed around as strings in the legacy's `MMDDYY` format because no one wrote a parser.
- The new service's enum for shipping methods includes legacy-specific values (`OVERNIGHT_RUSH_FLAG`) that exist only for the legacy's billing quirks.

Each of these is a small concession to integration convenience. Aggregated, they turn your new service into a thin veneer over the legacy — defeating the whole purpose of building something new. Worse, when the legacy is eventually retired, you discover that *removing* its concepts from your code requires changes everywhere.

Eric Evans (in *Domain-Driven Design*, 2003) named this risk and the defense against it: an **Anti-Corruption Layer** — a translator that sits between the new system and the foreign one, ensuring the new domain model is *never* contaminated by foreign concepts.

## How it works

> [!TIP]
> **ELI5.** Build one component whose only job is to call the legacy and translate. Your domain logic calls a clean API: `placeOrder(customer, items)`. The ACL receives this, translates it into the legacy's SOAP-and-magic-strings world, calls legacy, gets back a messy response, translates it into a clean Order, and returns it. Your domain logic never sees the legacy mess.

The ACL is a discrete component — usually a module, package, or microservice — that:

![Anti-Corruption Layer structure](../diagrams/svg/anti-corruption-layer.svg)

1. **Exposes a clean interface to your domain.** The new code calls methods that speak its own ubiquitous language: `Customer findByEmail(String email)`, `Order placeOrder(Cart cart)`. No legacy types or codes appear in these signatures.

2. **Translates domain calls into legacy calls.** Inside the ACL, the clean method bodies do the messy work: build the legacy SOAP request, set the obscure header fields, map domain concepts to legacy codes, etc.

3. **Translates legacy responses into domain types.** Take the legacy's response (with its 200 fields, 1970s date strings, mixed-case enum strings) and construct clean domain objects.

4. **Handles legacy-specific concerns** that your domain shouldn't see: caching to compensate for slow legacy queries, retries on legacy's flaky network behavior, mapping legacy error codes into domain exceptions, providing fallback behavior for legacy outages.

5. **Stays isolated.** Nothing outside the ACL imports legacy types. There's a hard architectural rule: legacy concepts may not appear in domain code, repository interfaces, controllers, or tests of domain logic.

### Placement in a microservice

A typical layout:

![ACL placement within a microservice](../diagrams/svg/acl-placement.svg)

The Caller hits the new service's clean API. The API delegates to the Domain (entities, business rules). The Domain talks to Repositories or outbound adapters. *Only one of those adapters* — the ACL — knows about the legacy. Every other adapter integrates with modern, healthy systems normally.

The architectural invariant: **the Domain never directly imports legacy types**. Static analysis or dependency rules (ArchUnit, deptrac, custom linters) can enforce this. When the legacy is eventually retired, you delete the ACL — and the Domain is unchanged.

### The ACL is intentionally disposable

A key cultural insight from Evans: the ACL is *expected* to be thrown away. Its existence is a sign of an integration that will eventually go away — when the legacy is retired, or when the upstream service is rewritten to provide a clean API. Treating the ACL as permanent infrastructure leads teams to over-engineer it.

In practice, the ACL is one of the most disposable components in the system. Its code quality should be "good enough to work and be understood" — not "production-grade beautiful." Most of its tests can be integration tests against a mocked legacy; its internal design can be straightforwardly procedural.

### ACL in microservice migration (strangler companion)

The ACL is the natural companion to the [Strangler Fig](strangler-fig.md) pattern. During strangulation, the new services often *still* need data from the legacy (which hasn't yet been decommissioned). Each new service that needs legacy data builds its own ACL — protecting itself from legacy concepts even though legacy is still there.

When the strangler finishes and legacy is decommissioned, all the ACLs are deleted at once. The new services remain clean.

### ACL across team boundaries (DDD context mapping)

Beyond legacy integration, the ACL pattern applies whenever your team integrates with another team's bounded context whose model differs from yours. Common scenarios:

- **Third-party APIs**: Stripe's API uses its own model (Charges, Customers, PaymentMethods, Subscriptions); your domain might think in terms of Orders, Invoices, Memberships. An ACL translates.
- **External services from other companies**: a shipping carrier's API uses Country-Code-2 vs Country-Code-3, weight in ounces vs grams, status codes with different semantics.
- **Cross-team service boundaries within a single company**: the "Orders" team's notion of `Order` may differ from the "Fulfillment" team's. An ACL on the consuming side prevents implicit coupling.

The first time you write code like `if (response.status == "PROCESSING_HOLD_AWAITING_REVIEW")`, you're skipping the ACL and pulling foreign concepts into your domain. That string belongs inside the ACL, mapped to your `OrderStatus.PENDING_REVIEW`.

### ACL vs Facade vs Adapter

The patterns overlap but have different intents:

- **Facade** simplifies a complex API; the underlying API is yours. Internal-facing.
- **Adapter** translates between two interfaces with no model mismatch — purely syntactic translation. Often automated by code-gen.
- **ACL** translates between two *domain models* — semantic translation that requires domain knowledge. Often the most complex of the three.

An ACL might be implemented *using* an adapter under the hood, but the ACL's job is the domain-model translation, not just the protocol/transport.

### Anti-patterns

- **No ACL, "we'll refactor later"** — never happens; legacy concepts metastasize.
- **ACL with leaky abstractions** — the ACL exposes legacy types in its public methods. Defeats the purpose.
- **Multiple ACLs in different places** — each component that talks to legacy reinvents the translation. Centralize.
- **ACL is shared across teams** — different teams need different translations for different contexts; trying to make one ACL serve all creates a new monolith.
- **ACL becomes the system of record** — when the ACL starts caching legacy data and serving it, it accidentally becomes a critical service. Be deliberate about this if it happens.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Translator-only ACL** | Pure stateless mapping between models. |
| **Cached ACL** | Adds caching to absorb legacy's latency/load. |
| **Active-fallback ACL** | Falls back to stored/derived data if legacy is unreachable. |
| **Async ACL** | Subscribes to legacy CDC events, exposes clean event stream to new services. |
| **Per-bounded-context ACL** | DDD strategic pattern: each bounded context owns its ACL into others. |
| **API Gateway as ACL** | Gateway provides clean external API by translating from messy internal services. |
| **Adapter (GoF)** | Syntactic translation only; no domain mapping. ACL is its domain-aware cousin. |

## When NOT to use

- **The upstream system already uses your domain model** — no corruption risk; just call directly.
- **A purely throwaway integration** (a script, a one-off batch) — overhead isn't worth it.
- **The "foreign" system is so trivial that one helper function suffices** — formal ACL is over-engineering.
- **You're going to rewrite both sides together** — no need to protect a model you're about to change.

---

## Real-world implementations

| Layer | Example |
|---|---|
| **Code-level ACL** | A dedicated package/module within a service (the common case). |
| **ACL as separate microservice** | A standalone "legacy adapter" service exposing a clean REST/gRPC API. |
| **ACL via API Gateway** | Gateway translates external clients' requests into internal microservice protocols (Stripe, Twilio's edge layer). |
| **ACL via event-stream adapter** | Debezium + transformation: legacy DB CDC → clean Kafka topic for new services. |
| **ACL via service-mesh filter** | Envoy filters that rewrite legacy headers/protocols into modern ones. |
| **Architectural enforcement** | ArchUnit (Java), deptrac (PHP), `boundaries` (Ruby) to enforce "no legacy types in domain." |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **eBay's marketplace re-platform** | Multi-year migration used ACLs to keep new services free of legacy V3 concepts. | ⚠ Discussed in QCon talks; specific implementations vary |
| **Capital One** | Bank's "cloud migration" effort used ACLs to wrap mainframe systems. | ✅ Verified — public AWS re:Invent talks |
| **Shopify** | Public engineering posts describe ACL-style modules wrapping legacy code during their modular-monolith migration. | ✅ Verified — Shopify Engineering blog |
| **Stripe** | Internal architecture uses ACL-like adapters at the boundary with banking-network legacy systems. | ⚠ Mentioned in engineering talks |
| **Almost every Heritage/Banking modernization** | The ACL is the standard pattern for talking to mainframe CICS / IMS / Db2 systems from microservices. | ✅ Verified — IBM, Accenture, Pivotal whitepapers |

---

## Further reading

- Eric Evans, *Domain-Driven Design* (2003), Ch 14 (Context Mapping) — the foundational treatment of the ACL pattern. The book itself is the right place to understand the intent.
- Microsoft Azure Architecture Center, *Anti-Corruption Layer pattern* — practical guidance.
- Chris Richardson, *microservices.io* — Anti-Corruption Layer in the microservices context.
- Vaughn Vernon, *Implementing Domain-Driven Design* (2013) — Ch on context mapping with concrete code examples.
- Sam Newman, *Monolith to Microservices*, Ch 4 — using ACLs during strangulation.
- Michael Feathers, *Working Effectively with Legacy Code* — code-level patterns that complement ACLs.

---

*Diagram sources: [`../diagrams/src/anti-corruption-layer.d2`](../diagrams/src/anti-corruption-layer.d2), [`../diagrams/src/acl-placement.d2`](../diagrams/src/acl-placement.d2).*
