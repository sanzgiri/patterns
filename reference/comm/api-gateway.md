# API Gateway

**Aliases:** Edge Gateway, Front Door
**Category:** Communication
**Sources:**
[Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/gateway) ·
[microservices.io](https://microservices.io/patterns/apigateway.html) ·
[ByteByteGo](https://github.com/ByteByteGoHq/system-design-101)

---

## Problem

> [!TIP]
> **ELI5.** Your apartment building has 50 small businesses inside. Without a lobby, every visitor has to wander the halls looking for the right one — and each business has to install its own front door, intercom, and security guard. Better: one shared lobby with a directory and security at the entrance.

When a system is decomposed into many services ([microservices](../arch/microservices.md)), clients face an unpleasant choice: either know about every service directly (their hostnames, ports, auth, retry behavior, versions), or accept that the system's complexity is *exposed* through its API surface. Both are bad. Direct connections mean every service must independently implement TLS termination, authentication, rate limiting, request logging, and CORS — and changing the service topology breaks every client. Every team reimplements the same cross-cutting concerns slightly differently.

Worse, a single client screen ("my dashboard") often needs data from 3–5 services. Without aggregation, the client must make 3–5 round trips itself — devastating on mobile networks. Pushing this into clients makes them brittle, version-sensitive, and slow.

## How it works

> [!TIP]
> **ELI5.** Put one server at the front door. All clients talk to it. It checks who they are, decides where to send their request, and sometimes calls several services and combines the answers before replying.

An API Gateway is a **single network entry point** that fronts a fleet of backend services. Every client request — web, mobile, third-party — passes through it. The gateway handles cross-cutting concerns once, in one place, and forwards to the appropriate backend.

![API Gateway structure](../diagrams/svg/api-gateway-structure.svg)

In the structure above, **Web**, **Mobile**, and **Partner** clients all converge on the **API Gateway** (orange). Inside the gateway live a stack of concerns that every request flows through: **auth/token validation**, **rate limiting**, **routing** (which backend handles this request, based on URL path/host/header), **aggregation** (optional — for endpoints that combine multiple services), and **logging/tracing** (correlation IDs injected, requests recorded). The gateway then forwards to the correct backend — `/orders/*` to the Orders Service, `/users/*` to the Users Service, and so on. The backends never see raw client traffic.

Two consequences follow. First, the gateway becomes the **place to enforce policy**: a security team can require all requests to carry a valid JWT *here*, without trusting every backend team to implement it correctly. A platform team can enforce rate limits and quotas centrally. SREs can flip a single config to enable mTLS to backends or rotate a TLS cert. Second, the **backend services can be ignorant of the outside world**: they speak plain HTTP/gRPC, listen on private network addresses, and trust that anything reaching them has already been authenticated.

The other major use case is **request aggregation** — collapsing what would be N client round trips into one:

![API Gateway aggregation](../diagrams/svg/api-gateway-aggregation.svg)

The client makes a single `GET /me/dashboard` request. The gateway, knowing this endpoint composes data from three services, fires three internal requests in parallel — `GET /users/123` to the Profile Service, `GET /orders?user=123` to Orders, `GET /recs?user=123` to Recommendations — collects the responses, merges them into a single JSON payload, and returns one response to the client. The client made **one round trip** instead of three, and got the data shaped exactly for its screen. The cost of each backend call has been moved from the public internet (slow) to the data-center LAN (fast).

This aggregation capability creates a temptation: stuff more and more business logic into the gateway, and you've recreated a centralized monolith routing all traffic. Two design rules push back against that. First, **the gateway should not own domain logic** — it composes and routes, but doesn't make business decisions; those belong in services. Second, the **[Backend for Frontend (BFF)](../comm/bff.md)** pattern explicitly splits the gateway by client type: a Web BFF for the web app, a Mobile BFF for mobile, a Partner BFF for B2B — each owned by the team that owns that client. This keeps any one gateway from becoming a contention point and gives clients APIs shaped for their needs.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Gateway Routing** | Just URL-based forwarding; no aggregation or auth. The minimal version. |
| **Gateway Aggregation** | Adds backend fan-out and response merging — the second diagram above. |
| **Gateway Offloading** | Moves cross-cutting concerns (TLS termination, compression, auth) from services to gateway. |
| **Backend for Frontend (BFF)** | One gateway per client type — avoids the "one gateway tries to please everyone" trap. |
| **Service Mesh** | Solves a *different* problem — service-to-service traffic inside the cluster. Often paired: API Gateway for north-south traffic, service mesh for east-west. |
| **Reverse Proxy** | The minimal version of an API gateway — just routing and possibly TLS. NGINX, HAProxy, Envoy. |

## When NOT to use

- **A single backend service.** A reverse proxy or load balancer is enough; the "gateway" framing adds nothing.
- **When latency-sensitive callers are inside the same trust boundary as the services.** Don't put a gateway between two internal services — use a service mesh or direct calls.
- **When the gateway is on the verge of becoming "the place where things go wrong."** If half your incidents are gateway misconfigurations or capacity issues, you've probably stuffed too much into it. Split into BFFs.

---

## Real-world implementations

| Tool | Notes |
|---|---|
| **Kong** | Open-source API gateway built on NGINX; large plugin ecosystem; managed offering (Konnect). |
| **AWS API Gateway** | Managed gateway on AWS; integrates natively with Lambda, IAM, Cognito. |
| **Apigee** (Google Cloud) | Enterprise-focused; strong analytics and developer-portal features. |
| **Azure API Management** | Microsoft's managed gateway; equivalent to AWS API Gateway. |
| **Tyk** | Open-source; high-performance; good for B2B / partner APIs. |
| **Envoy + custom config / Istio Ingress / Contour / Emissary** | Gateway built on the Envoy proxy; the modern open-source default for Kubernetes. |
| **Spring Cloud Gateway** | JVM-native; works well with Spring Boot microservices. |
| **Express Gateway / NestJS gateway** | Node-native; commonly used for BFF patterns. |
| **Netflix Zuul** | The original microservice gateway; superseded internally by Envoy-based stack but still influential. |

## Companies using it (notable examples)

| Company | Use | Status |
|---|---|---|
| **Netflix** | Zuul (and now their Envoy-based successor) fronts essentially all client traffic; aggregates calls for the device UIs. | ✅ Verified — [*Open Sourcing Zuul 2*, Netflix Tech Blog, 2018](https://netflixtechblog.com/open-sourcing-zuul-2-82ea476cb2b3) |
| **Amazon (AWS)** | Operates one of the largest production gateways; sells it as AWS API Gateway. | ✅ Verified — [AWS docs](https://docs.aws.amazon.com/apigateway/) |
| **Lyft** | Originated Envoy; runs it as edge proxy + internal service mesh. | ✅ Verified — [*Announcing Envoy*, Lyft Engineering, 2016](https://www.lyft.com/blog/posts/announcing-envoy) |
| **Airbnb** | Uses a custom gateway / BFF layer to aggregate microservice data for web and mobile. | ⚠ Discussed in talks; not re-verified by fetch |
| **SoundCloud** | The team that coined "Backend for Frontend" (Phil Calçado) at SoundCloud; their gateway was BFF-style from early on. | ✅ Verified — [Sam Newman, *BFF*, 2015, citing SoundCloud origin](https://samnewman.io/patterns/architectural/bff/) |
| **Spotify** | Operates client-specific gateways for web and mobile. | ⚠ Discussed in talks; not re-verified |

**⚠ marks claims widely known industry-wide but not re-verified by primary-source fetch.**

---

## Further reading

- Sam Newman, *Backends for Frontends* — [samnewman.io/patterns/architectural/bff/](https://samnewman.io/patterns/architectural/bff/).
- Chris Richardson, *Microservices Patterns*, Ch 8 — API gateway + BFF treatment.
- Microsoft Azure Architecture Center, *Gateway Routing / Aggregation / Offloading* — three separate but related pattern docs.
- *Building Microservices* (Sam Newman), Ch 4 — discusses where the gateway fits in the broader service-communication picture.

---

*Diagram sources: [`../diagrams/src/api-gateway-structure.d2`](../diagrams/src/api-gateway-structure.d2), [`../diagrams/src/api-gateway-aggregation.d2`](../diagrams/src/api-gateway-aggregation.d2).*
