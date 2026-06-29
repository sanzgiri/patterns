# Microservice Chassis

**Aliases:** Service Chassis, Microservice Framework, Application Framework, Service Runtime
**Category:** Microservices / Developer productivity
**Sources:**
[Chris Richardson — microservices.io: Microservice Chassis](https://microservices.io/patterns/microservice-chassis.html) ·
Sam Newman, *Building Microservices* (2nd ed.) ·
Discussion in Spring Boot, Micronaut, Quarkus, Dropwizard documentation

---

## Problem

> [!TIP]
> **ELI5.** Every microservice needs the same plumbing: logging, metrics, tracing, config, health checks, circuit breakers, graceful shutdown, service discovery. Writing all that from scratch in every service is slow, inconsistent, and error-prone. A **chassis** is a framework — like Spring Boot or Quarkus — that provides all this plumbing as a library you depend on. Your service inherits production-grade infrastructure for free, and just adds business logic.

A microservice in production needs a *lot* of cross-cutting infrastructure beyond its business logic:

- **Structured logging** with correlation IDs, automatic field injection, log-level control.
- **Metrics** for request count, latency, error rate, JVM/process statistics, custom business metrics — exposed via Prometheus or StatsD.
- **Distributed tracing** with OpenTelemetry, automatic span creation for inbound/outbound requests, trace-context propagation.
- **Configuration** loaded from environment variables, files, and remote config services, with hot-reload support.
- **Service discovery** integration — register on startup, deregister on shutdown, client-side lookup.
- **Resilience primitives** — circuit breaker, retry, timeout, bulkhead — wrapping outbound calls.
- **Authentication / authorization** — JWT verification, OAuth client flow, mTLS, role-based access.
- **Health and readiness endpoints** for orchestrators (Kubernetes, ECS, Nomad).
- **Graceful shutdown** — handle `SIGTERM`, stop accepting new requests, drain in-flight, deregister, exit.
- **Request lifecycle middleware** — request logging, error handling, correlation-ID injection.

Building all this from scratch in every service is enormous duplicated effort. Worse, the *result* is bimodal: a few teams build excellent infrastructure; most build mediocre or broken versions; the platform team can't enforce a baseline. Cross-cutting concerns are precisely the things that *should* be the same everywhere — they're the bread and butter of operational excellence.

The **Microservice Chassis** pattern is the answer: a shared framework or library that provides production-grade cross-cutting concerns out of the box. New services inherit them by simply depending on the chassis.

## How it works

> [!TIP]
> **ELI5.** Build (or pick) a framework that wraps every microservice in standard infrastructure. The framework boots up your service, sets up logging/metrics/tracing/health checks, handles signals, registers with discovery — all without your code doing anything. You write a few handlers and the chassis takes care of the rest. When the platform team improves the chassis (adds a metric, fixes a bug), every service that bumps the dependency gets the improvement.

A chassis provides a programming model where the developer writes only business code, and the chassis handles cross-cutting concerns:

![Microservice Chassis: what the framework provides](../diagrams/svg/microservice-chassis.svg)

In a Spring Boot service, for example, the developer writes:

```java
@RestController
class OrderController {
    @PostMapping("/orders")
    Order placeOrder(@RequestBody Cart cart) { ... }

    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable String id) { ... }
}
```

That's the entirety of new code for a simple service. The Spring Boot chassis automatically provides:

- HTTP server (embedded Tomcat / Netty).
- Request routing and parameter binding.
- JSON serialization (Jackson).
- Logging with correlation IDs (via Sleuth / Micrometer).
- Metrics endpoint (`/actuator/prometheus`).
- Health and readiness endpoints (`/actuator/health`, `/actuator/readiness`).
- OpenTelemetry tracing (via Micrometer + OTel exporter).
- Config from `application.yml`, environment, and remote sources.
- Graceful shutdown on `SIGTERM`.
- Circuit breaker via Resilience4j integration.
- JWT validation via Spring Security.

The developer writes ~50 lines of business code and inherits hundreds of lines of production-grade infrastructure. This is the chassis pattern's value proposition.

### Chassis vs Template

The chassis is often confused with the [Service Template](service-template.md). They're complementary:

- A **template** is a one-time scaffold that produces files (Dockerfile, CI config, layout).
- A **chassis** is a runtime dependency that produces behavior (logging, metrics, tracing).

Most mature organizations use both: the template scaffolds the repository structure; the scaffolded service depends on the chassis library. Improvements to cross-cutting *behavior* propagate via chassis version bumps — far easier than re-running the template across hundreds of repositories.

### What makes a good chassis

A few qualities distinguish well-designed chasses:

- **Sensible defaults.** Out of the box, a chassis-based service should ship production-quality logs, metrics, and tracing with zero configuration.
- **Extension points.** When the default isn't right, customization is easy and well-documented.
- **Composable.** Features can be enabled/disabled independently — don't force every service to take everything.
- **Backward-compatible upgrades.** Bumping the chassis version shouldn't break existing services. Spring Boot's careful versioning is a good example.
- **Observable.** The chassis itself exposes metrics about its own behavior (e.g., circuit-breaker state, connection-pool stats).
- **Language-idiomatic.** Use the language's normal patterns rather than a foreign DSL. Java chasses use annotations; Go chasses use middleware functions and context; Python chasses use decorators.

### Chassis trade-offs

The chassis approach has real costs:

- **Language lock-in.** A chassis is specific to one language and often one framework. A polyglot organization needs multiple chasses, often diverging in feature parity. This is the main reason large polyglot orgs (Google, Netflix) eventually invested in **service mesh** (language-agnostic sidecar proxies) for many of the same concerns.
- **Coupling to a vendor or community.** Choosing Spring Boot means accepting its decisions, release cadence, and ecosystem. Migrating away is expensive.
- **Bloat.** Chasses often include features you don't need; the binary/container is larger than necessary.
- **Magic / debugging difficulty.** Auto-configuration and convention-over-configuration make simple things simple, but make troubleshooting harder when defaults are wrong.
- **Slow startup.** Reflection-heavy chasses (especially Spring Boot in the pre-AOT era) can take 10–30 seconds to start, problematic for serverless or CI.

The last point drove a generation of chassis innovation: Micronaut, Quarkus, and Spring Boot's GraalVM native mode all dramatically reduce startup time by moving work to build-time.

### Chassis vs Service Mesh

For language-agnostic cross-cutting concerns, the **service mesh** (Istio, Linkerd) is the modern alternative. A sidecar proxy (Envoy) handles mTLS, retries, circuit breakers, observability — *outside* the service process, regardless of language.

The chassis and the mesh aren't mutually exclusive. Typical division of labor:

- **Mesh handles** network-level concerns: mTLS between services, request retries, traffic splitting, observability of network requests, rate-limiting at the network edge.
- **Chassis handles** in-process concerns: domain-aware logging, business metrics, ORM, async processing, configuration, lifecycle hooks.

Many production architectures use both. The chassis becomes lighter when the mesh handles network concerns; the mesh doesn't try to do things that require in-process context.

### Building vs buying a chassis

Should you build your own chassis or use one off the shelf?

- **Start with off-the-shelf** (Spring Boot, Quarkus, Micronaut, Dropwizard for Java; Gin or Echo + Go-Kit for Go; NestJS for Node; FastAPI for Python). The mature ones have years of operational hardening.
- **Add an organization-specific "starter" or extension** that wraps the off-the-shelf chassis with company-specific opinions (your logging conventions, your auth integration, your discovery client). This is what most large orgs do.
- **Pure custom chassis** is rare and usually a mistake — you re-invent solved problems and miss security fixes.

The sweet spot: a thin organization-specific layer over a well-established underlying framework. Spotify's Apollo framework, LinkedIn's Rest.li, and Netflix's various internal extensions of Spring Boot all follow this pattern.

### The long-term cost

A chassis is a long-term commitment. Once 200 services depend on `your-chassis-v1`, upgrading them all is a multi-quarter program. Versioning, migration paths, and deprecation policy become critical. The platform team owns this responsibility forever — a chassis is not a project, it's a product.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Off-the-shelf chassis** | Spring Boot, Quarkus, Micronaut, Dropwizard, NestJS, FastAPI. |
| **Org-extended chassis** | Off-the-shelf + thin company-specific wrapper layer. |
| **Custom-built chassis** | Pure in-house; rare and usually a mistake. |
| **Polyglot chasses** | One chassis per primary language, with feature parity as goal. |
| **Mesh-augmented chassis** | Chassis + service mesh splits responsibilities. |
| **AOT-compiled chassis** | Quarkus, Spring Native — for fast startup and low memory. |
| **Serverless chassis** | Lambda Powertools, Azure Functions Worker — chassis for FaaS. |

## When NOT to use

- **Single service / monorepo with shared code already** — chassis overhead doesn't pay off.
- **Highly heterogeneous services** that share little cross-cutting concern.
- **Prototypes and one-offs** — production-grade chassis is over-engineering.
- **Latency-critical low-level services** where chassis overhead matters and you need full control.
- **Polyglot orgs without resources to maintain multiple chasses** — consider service mesh instead.

---

## Real-world implementations

| Chassis | Language | Notes |
|---|---|---|
| **Spring Boot** | Java/Kotlin | The dominant Java chassis; vast ecosystem. |
| **Micronaut** | Java/Kotlin/Groovy | AOT, fast startup, low memory. |
| **Quarkus** | Java/Kotlin | "Supersonic, Subatomic Java"; GraalVM native. |
| **Dropwizard** | Java | Older, simpler, popular in finance. |
| **Helidon** | Java | Oracle's chassis; SE and MP variants. |
| **Vert.x** | Java/Kotlin/Groovy | Reactive, polyglot. |
| **Akka HTTP / Lagom** | Scala/Java | Akka-based reactive chassis. |
| **NestJS** | TypeScript/Node.js | Heavily inspired by Spring; opinionated. |
| **FastAPI** | Python | Modern async chassis with auto OpenAPI. |
| **Go-Kit, Goa, Kratos** | Go | Multiple competing options. |
| **ASP.NET Core** | C#/.NET | Microsoft's chassis; production-grade. |
| **.NET Aspire** | C#/.NET | Microsoft's newer "cloud-native chassis" framework. |
| **Phoenix** | Elixir | Erlang VM chassis; popular in finance/realtime. |
| **Lagom** | Scala/Java | Specifically aimed at microservice chassis. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Pivotal / VMware customers (Bank of America, Comcast, Walmart Labs)** | Spring Boot is the de-facto chassis; thousands of services. | ✅ Verified — Spring case studies |
| **Netflix** | Originally heavy Spring Boot users; extended internally with Hystrix, Ribbon, Eureka. | ✅ Verified — Netflix Tech Blog |
| **LinkedIn** | Rest.li is LinkedIn's open-source chassis for Java services. | ✅ Verified — [Rest.li on GitHub](https://github.com/linkedin/rest.li) |
| **Spotify** | Apollo (Java) for backend services. | ✅ Verified — [Apollo on GitHub](https://github.com/spotify/apollo) |
| **Red Hat** | Quarkus is Red Hat's flagship chassis. | ✅ Verified — Quarkus is a Red Hat project |
| **Google Cloud customers via Go-Kit / gRPC** | gRPC + interceptors function as a chassis in many Go services. | ✅ Verified — gRPC docs |
| **Microsoft / .NET ecosystem** | ASP.NET Core + Aspire is the standard chassis. | ✅ Verified — Microsoft docs |
| **Many enterprise platform teams** | Org-extended Spring Boot starters (custom auth, logging, metrics integration). | ✅ Verified — common pattern documented at QCon, KubeCon |

---

## Further reading

- Chris Richardson, *microservices.io* — Microservice Chassis pattern entry.
- Sam Newman, *Building Microservices* (2nd ed.), Ch on developer productivity — chassis vs template trade-offs.
- Spring Boot reference documentation — the most-mature open chassis.
- Micronaut and Quarkus documentation — modern AOT alternatives.
- Brendan Burns, *Designing Distributed Systems* — container patterns that complement chassis features.
- *Team Topologies*, Skelton & Pais — discusses platform-team responsibility for chassis.
- "Service Mesh vs Microservice Chassis" — multiple comparative blog posts (Buoyant, Solo.io) on dividing responsibilities.

---

*Diagram sources: [`../diagrams/src/microservice-chassis.d2`](../diagrams/src/microservice-chassis.d2), [`../diagrams/src/template-vs-chassis.d2`](../diagrams/src/template-vs-chassis.d2).*
