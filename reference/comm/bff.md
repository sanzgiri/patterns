# Backends for Frontends (BFF)

**Aliases:** BFF, Client-Specific Gateway, Experience API
**Category:** Communication / Architectural
**Sources:**
[Sam Newman, *Backends For Frontends* (2015)](https://samnewman.io/patterns/architectural/bff/) ·
[Phil Calcado, *The Back-end for Front-end Pattern (BFF)*, SoundCloud (2015)](https://philcalcado.com/2015/09/18/the_back_end_for_front_end_pattern_bff.html) ·
[Microsoft Azure Architecture Center — Backends for Frontends](https://learn.microsoft.com/en-us/azure/architecture/patterns/backends-for-frontends) ·
[Chris Richardson — microservices.io: BFF](https://microservices.io/patterns/apigateway.html)

---

## Problem

> [!TIP]
> **ELI5.** Your microservices speak a generic API. But your web app, mobile app, and IoT device each want different data shapes, different protocols, and different latency-vs-payload trade-offs. If they all share one API, none of them gets exactly what it wants — and the shared API becomes a battleground of conflicting requirements. The **BFF** pattern: give each client its own gateway, tailored to its needs. Each frontend team owns its backend.

A microservices architecture exposes services with stable, generic APIs ("here's a Customer resource, here's an Order resource"). But the *clients* of those APIs are increasingly diverse:

- **Web apps** typically want full, nested data — they have bandwidth, GPUs to render, and benefit from receiving complete object graphs.
- **Mobile apps** want trimmed, flat payloads — bandwidth is precious, parsing is expensive, and battery cost of multiple round-trips is real.
- **IoT devices** want tiny binary payloads, often over MQTT/CoAP rather than HTTP, with batching and deduplication built in.
- **Voice / chatbot interfaces** want denormalized natural-language-friendly responses.
- **Partner B2B integrations** want stable contracts with strict versioning and SLAs.
- **Internal admin tools** want full data with bulk operations.

If all of these share one generic API, you get:

- **Bloated responses** for the weakest clients (mobile downloads 10× more data than it needs).
- **Multiple round-trips** for clients that wanted aggregated data.
- **"Design by committee"** — every client team has veto power over API changes; nothing ships quickly.
- **Coupling between unrelated teams** — the web team's needs block the mobile team's release.
- **Cross-cutting concerns implemented poorly** — the API has to support web's auth and mobile's auth and partner's auth simultaneously.

Sam Newman and Phil Calcado articulated the **Backends for Frontends** pattern at SoundCloud in 2015 as the answer: instead of one shared API, give each client its own gateway — owned by the client team, tailored to its experience, free to evolve independently.

## How it works

> [!TIP]
> **ELI5.** Build one small gateway per client experience: a Web BFF, a Mobile BFF, an IoT BFF, a Partner BFF. Each BFF is owned by the team building that client. It aggregates calls to the backend microservices and returns exactly the shape that client needs. Backend services stay generic; clients get tailored experiences.

The structure is straightforward:

![BFF: one tailored gateway per client](../diagrams/svg/bff-pattern.svg)

Without BFF, all clients hit one generic API which compromises for everyone. With BFF, each client has its own backend that:

- **Aggregates** calls to multiple backend services into one response.
- **Trims** data to what the client actually needs.
- **Reshapes** the JSON (or binary, or GraphQL) into the client's preferred form.
- **Uses the client's preferred protocol** — HTTP/JSON for web, gRPC or binary for mobile, MQTT for IoT.
- **Handles client-specific cross-cutting concerns** — different auth methods, different rate limits, different caching strategies.
- **Owns its own deployment cadence** — released with the client, not with backend services.

Critically, each BFF is **owned by the team that builds the client**:

![BFF ownership: each client team owns its BFF](../diagrams/svg/bff-ownership.svg)

The web team owns the web app *and* the web BFF, deploying them together. The mobile team owns iOS, Android, *and* the mobile BFF. Partner integrations live in a partner BFF, owned by the team that contracts with partners.

This ownership model is the deepest insight in the BFF pattern. It's not just a technical reorganization; it's a Conway's-law-aware reshaping of the org. Teams that own the client experience also own the backend that serves it.

### What the BFF is allowed to do

The BFF should:

- **Aggregate**: call multiple backend services, combine results.
- **Project / trim**: select only fields the client needs.
- **Reshape**: nest, flatten, rename to suit the client.
- **Translate**: between protocols (gRPC ↔ REST, REST ↔ GraphQL).
- **Cache**: client-specific caching strategies.
- **Apply client-specific auth and rate limits**.
- **Handle client-specific resilience needs** (mobile may need more aggressive retry due to network unreliability).

The BFF should *not*:

- **Contain core business logic** — that belongs in backend services.
- **Own data** — BFFs are stateless; data lives in services.
- **Make policy decisions** affecting backend services — e.g., the mobile BFF shouldn't change how orders are validated.
- **Be reused across multiple client experiences** — that defeats the purpose. If you have 3 mobile apps that need the same BFF, then either make one mobile BFF, or accept that they're really one experience.

The discipline of keeping BFFs thin and aggregation-only is important. A BFF that grows business logic becomes a duplicate domain layer that drifts from backend services and creates inconsistencies.

### BFF vs API Gateway

The BFF is essentially **a per-client API gateway**:

- A traditional [API Gateway](api-gateway.md) is a single shared entry point for all clients, owned by a platform team.
- A BFF is one of multiple gateways, each tailored to a specific client experience, owned by the team building that client.

You can use both: a platform-owned gateway at the edge (auth, rate-limit, routing to BFFs); per-team BFFs behind it (aggregation, reshaping). Many large architectures look exactly like this.

### BFF and GraphQL

GraphQL has a particular relationship to BFF:

- A **GraphQL server can serve as a BFF**: it exposes a flexible, client-driven query interface. Many web BFFs are GraphQL servers.
- A GraphQL **federation gateway** can also be a BFF or sit between BFFs and backend services.
- Some argue GraphQL eliminates the need for BFFs (since clients can request exactly what they want). In practice, you usually still want per-client GraphQL servers because the schema, auth, and aggregation logic differ.

The right architecture varies. GraphQL-as-BFF is common; GraphQL-federation-as-shared-gateway with per-client GraphQL BFFs is also common.

### Common pitfalls

- **BFF becomes a monolith**: one team builds "the BFF" for all clients; the original problem returns.
- **Business logic in BFF**: BFF duplicates validation, calculation, or rule logic from backend services. Drift causes bugs.
- **BFF owns data**: BFF starts caching or even storing data; suddenly it's a critical service.
- **Per-client BFF for clients that are really the same**: iOS and Android usually share a BFF; web and admin usually have separate BFFs.
- **No platform team for cross-BFF concerns**: every BFF reinvents auth, observability, deployment. Mitigation: BFF template / chassis owned by platform team.
- **Backend services start to know about BFFs**: backend APIs accommodate BFF needs specifically, coupling backend evolution to client evolution.

### When BFFs are most valuable

BFFs work best when:

- Multiple distinct client experiences with materially different needs.
- Client teams have engineering capacity to own a backend.
- Backend services are reasonably stable (so BFFs can aggregate them without constant breakage).
- Org structure supports vertical (client-experience) teams.

BFFs are overkill when:

- Only one client experience (just call the API directly).
- Client teams are too small to own a backend.
- Backend services are changing so rapidly that BFFs spend all their time keeping up.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Web BFF (REST/GraphQL)** | Returns rich, nested data; often GraphQL-based. |
| **Mobile BFF** | Trimmed, batched, often binary-encoded responses. |
| **IoT BFF** | MQTT/CoAP, tiny payloads, server-driven push. |
| **Partner BFF** | Strict contracts, versioning, SLAs, separate auth. |
| **Per-channel BFF** | Voice, chat, smart-TV each get their own. |
| **GraphQL-as-BFF** | Flexible query interface as the BFF. |
| **Federated GraphQL** | Single gateway federating multiple GraphQL services (often per-team). |
| **API Gateway** | Single shared gateway — the predecessor pattern BFF supersedes for client-specific needs. |
| **Shared library / SDK** | An alternative for some concerns; client SDKs do BFF-like aggregation client-side. |

## When NOT to use

- **Single client experience** — no need for tailoring.
- **Tiny org without team capacity** to own per-client backends.
- **Highly homogeneous clients** — if web/mobile/partner can all use the same API, don't fragment.
- **Backend services unstable** — BFFs become a tax on every backend change.
- **Tight backend coupling needed** — if backends and clients are tightly coupled by design, BFF doesn't add value.

---

## Real-world implementations

| Tech | BFF role |
|---|---|
| **Node.js / Express / Fastify** | Common BFF stack; lightweight, async, JavaScript-shared with frontend. |
| **Next.js API routes / app router** | Effectively a BFF for the same Next.js frontend. |
| **Apollo GraphQL Server** | GraphQL-as-BFF; widely adopted. |
| **Hasura / PostGraphile** | Auto-generated GraphQL BFFs over PostgreSQL. |
| **Spring Boot + Spring Cloud Gateway** | Java-based BFF in mixed-stack orgs. |
| **gRPC-Web BFF** | Translates between gRPC backends and HTTP/JSON web clients. |
| **AWS AppSync** | Managed GraphQL BFF on AWS. |
| **Azure API Management with per-client policies** | Approximation of BFF with managed gateway. |
| **Kong, Tyk, KrakenD** | API gateways used as BFFs with per-client configurations. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **SoundCloud** | Coined the term; published the essay describing internal use. | ✅ Verified — [Phil Calcado's blog post](https://philcalcado.com/2015/09/18/the_back_end_for_front_end_pattern_bff.html) |
| **Netflix** | Per-device BFFs ("device experience APIs") — TVs, mobile, web each have tailored backends. | ✅ Verified — Netflix Tech Blog series |
| **Spotify** | Multiple BFFs per client type; described in engineering posts. | ✅ Verified — Spotify Engineering blog |
| **Airbnb** | GraphQL BFFs for web and mobile experiences. | ✅ Verified — Airbnb Engineering blog |
| **Shopify** | Storefront API + Admin API are effectively BFFs for different audiences. | ✅ Verified — Shopify dev docs |
| **GitHub** | GraphQL API for newer integrations is a BFF-style layer over internal services. | ✅ Verified — GitHub GraphQL API docs |
| **Twitter / X** | Per-platform BFFs for web vs mobile vs partner. | ⚠ Discussed in conference talks |

---

## Further reading

- Sam Newman, *Pattern: Backends For Frontends* (2015) — the foundational essay. [samnewman.io/patterns/architectural/bff](https://samnewman.io/patterns/architectural/bff/).
- Phil Calcado, *The Back-end for Front-end Pattern (BFF)* (SoundCloud, 2015) — the canonical case study.
- Microsoft Azure Architecture Center, *Backends for Frontends pattern*.
- Sam Newman, *Building Microservices* (2nd ed., 2021) — Ch covering BFF + Gateway interplay.
- *GraphQL: The Documentary* (Apollo) — historical context on how GraphQL relates to BFF.
- Multiple Netflix Tech Blog posts on device-specific backend APIs.
- Brendan Burns, *Designing Distributed Systems* — BFF in the context of broader patterns.

---

*Diagram sources: [`../diagrams/src/bff-pattern.d2`](../diagrams/src/bff-pattern.d2), [`../diagrams/src/bff-ownership.d2`](../diagrams/src/bff-ownership.d2).*
