# Ambassador

**Aliases:** Outbound Proxy, Out-of-Process Client, Client-side Sidecar
**Category:** Communication
**Sources:**
[Microsoft Azure Architecture Center — Ambassador pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/ambassador) ·
[Burns et al., *Design patterns for container-based distributed systems* (HotCloud 2016)](https://www.usenix.org/system/files/conference/hotcloud16/hotcloud16_burns.pdf) ·
Discussion in Brendan Burns, *Designing Distributed Systems* (O'Reilly)

---

## Problem

> [!TIP]
> **ELI5.** Your app needs to call 5 external APIs (Stripe, Twilio, SendGrid, an analytics service, a vendor's webhook). Each has its own auth, rate limits, retry rules, and quirks. Adding all that logic into your app — in your app's language — is painful and repetitive. The **ambassador** is a dedicated proxy process that sits next to your app and handles all the external-call complexity. Your app just makes plain calls to `localhost`; the ambassador takes care of the rest.

A microservice often needs to call **external services** — APIs run by other companies, third-party SaaS products, partner systems, or services in other regions. Each external service brings its own integration burden:

- **Authentication**: API key, OAuth flow with token refresh, mTLS, basic auth, signed requests.
- **Rate limiting**: each service has its own quotas; your app needs to respect them.
- **Retry policy**: each service has different idempotency guarantees and rate-limit behaviors; retries must be tailored.
- **Circuit breaking**: detecting that an external service is down and stopping calls until it recovers.
- **Caching**: many external responses are stable enough to cache locally.
- **Observability**: latency and error metrics per external dependency.
- **Failover**: secondary endpoints, alternate regions, fallback providers.
- **Protocol translation**: external service uses gRPC; your app speaks REST; or vice versa.

Implementing all this *inside* the main app means:

- The app's code is bloated with integration concerns unrelated to its business logic.
- Each language stack reimplements the same patterns.
- Changing retry policy for Stripe requires rebuilding and redeploying the entire app.
- Cross-team patterns (the platform team has expertise in retry/rate-limit strategy) are blocked from contributing to language-specific code.

The **Ambassador** pattern moves all this complexity into a dedicated sidecar process. The main app calls the ambassador over `localhost` with simple, in-language API calls; the ambassador handles the gnarly external-world integration.

## How it works

> [!TIP]
> **ELI5.** Deploy a small proxy process next to your app (in the same pod, same VM, same network namespace). The proxy is the *only* thing that calls external services. Your app calls the proxy on `localhost`. The proxy adds authentication tokens, handles retries on rate-limit responses, opens circuit breakers when external services are down, caches responses, and emits metrics. Your app code stays simple — it never sees OAuth tokens, retry logic, or 429 responses.

The ambassador is a specialized [sidecar](sidecar.md), focused on outbound calls to external services:

![Ambassador pattern](../diagrams/svg/ambassador-pattern.svg)

The main application makes simple calls to the ambassador over `localhost`:

```python
# main app code — simple, no auth/retry/error-handling
response = requests.post("http://localhost:9001/stripe/v1/charges",
                         json={"amount": 1000, "customer": cust_id})
```

The ambassador receives the call, then:

1. **Looks up the routing rule.** "`/stripe/*` goes to `api.stripe.com`."
2. **Adds authentication.** Loads the OAuth token (refreshed in the background), adds the `Authorization` header.
3. **Applies retry policy.** Posts to Stripe. If response is 429 (rate limited), waits per Stripe's documented backoff. If 5xx, retries with exponential backoff.
4. **Applies circuit breaker.** If recent error rate is too high, fails fast without calling Stripe.
5. **Caches** (where appropriate) — GET responses with cache headers, idempotent POSTs with idempotency keys.
6. **Emits metrics.** Stripe latency, error rate, retry count, circuit state.
7. **Returns the response** to the main app on `localhost`.

The main app sees a single clean call. It doesn't know about Stripe's specifics, doesn't manage OAuth tokens, doesn't implement retries. If Stripe's policies change tomorrow, only the ambassador's config changes — the app stays the same.

### Ambassador vs Sidecar vs Service Mesh Sidecar

These overlap; the distinctions are about scope:

- **Sidecar**: any helper container co-located with the main app. The umbrella concept.
- **Ambassador**: a sidecar specifically focused on **outbound** calls to **external** services. Scope is narrower than a full service-mesh sidecar.
- **Service-mesh sidecar** (Envoy/Linkerd): handles both inbound and outbound *and* internal mesh traffic, plus traffic policies, mTLS, traces, etc. Much broader scope.

If you already run Istio or Linkerd, you may not need a separate ambassador — the mesh's egress filters can do many of the same things. If you don't have a mesh, an ambassador is a lightweight focused alternative for the outbound-only case.

### When ambassadors are valuable

The ambassador shines in specific scenarios:

- **Apps in a niche language** calling well-known external services. A data-science app in Python can use a Go ambassador for Stripe — getting Go's better retry libraries without porting them to Python.
- **Multiple apps needing the same external integration.** Build the ambassador once; share it across teams. The Stripe-integration ambassador serves the orders, billing, and refunds apps identically.
- **Externalizing security concerns.** API keys live in the ambassador (with its own secret store), not in every app. App can be open-source while keys stay private.
- **Centralizing policy.** "Increase Stripe retry budget" is one ambassador change, not N app changes.
- **Testing.** A test-mode ambassador can serve canned responses without external calls. App doesn't need a test mode of its own.
- **Multi-region failover.** Ambassador knows which region's Twilio endpoint is healthy; routes accordingly. App is region-agnostic.

### Ambassador limitations

- **Latency.** Every external call goes through an extra process; +1ms on the local network. For most external calls (with their own 50–500ms latency) this is negligible.
- **Operational complexity.** Another process to deploy, version, monitor.
- **Single point of failure within the pod.** If the ambassador crashes, the app can't reach external services. Standard solution: pod restart, plus app-side timeouts.
- **Limited to outbound.** Inbound concerns (auth on requests *to* your service) are a different pattern (gateway, mesh inbound sidecar).

### Real-world manifestations

The ambassador pattern has manifested in many forms over the years:

- **AWS X-Ray daemon** as a sidecar that batches and forwards traces to AWS — a specialized ambassador for the observability backend.
- **Cloud SQL Proxy** — runs as a sidecar; the app connects to `localhost:5432`; the proxy handles auth, mTLS, and connection routing to Cloud SQL.
- **Stripe's recommended setup for high-volume merchants**: an in-process or sidecar layer that handles idempotency keys, retries, and rate limiting before calling Stripe APIs.
- **Vault Agent** — though primarily a secret sidecar, also handles outbound auth to Vault on the app's behalf.
- **Dapr** — Dapr's outbound bindings (calling Twilio, SendGrid, etc.) function as ambassadors with a standardized protocol.

The pattern is everywhere; the name "ambassador" isn't always used, but the structure recurs.

### Implementation considerations

- **Connection model.** Usually the app calls `http://localhost:port/` and the ambassador maps routes to external services. Alternatives: Unix domain sockets, gRPC streams.
- **Configuration.** The ambassador needs to know route → endpoint mappings, retry policies, circuit-breaker thresholds, cache TTLs. Usually static config or pulled from a config service.
- **Secret management.** API keys / OAuth credentials live with the ambassador. Often loaded from Vault, AWS Secrets Manager, or K8s secrets.
- **Observability.** Ambassador emits metrics labeled by external service; traces should propagate from app through ambassador to external (if the external supports trace headers).
- **Lifecycle.** Sidecar lifecycle — starts before app, terminates after. Healthchecks ensure the ambassador is reachable before the app is marked ready.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Generic outbound proxy** | The classic ambassador; handles N external services. |
| **Single-service ambassador** | Dedicated to one external service (e.g., Stripe-only). |
| **Test-mode ambassador** | Serves canned responses; substitutes for the real external service in tests. |
| **Region-failover ambassador** | Selects healthy regional endpoint; app is region-agnostic. |
| **Cloud-service proxy** | Cloud SQL Proxy, Bigtable Proxy — specialized ambassadors for managed services. |
| **Service mesh egress** | Mesh sidecar's egress feature subsumes the ambassador role when mesh is present. |
| **API gateway (as outbound aggregator)** | Different scope — gateway is usually inbound-facing for clients. |
| **Dapr binding** | Dapr's outbound bindings standardize ambassador-like calls across languages. |

## When NOT to use

- **One simple external service, called from one app, in a homogeneous language stack** — overhead of an extra process isn't worth it; an in-app library is simpler.
- **Apps already using a service mesh** — mesh egress handles most ambassador concerns.
- **Latency-critical paths** where the localhost hop matters.
- **One-off scripts / batch jobs** without long-running processes.
- **External services with no client-side complexity** (auth, retry, rate limit) — a plain library call is enough.

---

## Real-world implementations

| System | Role |
|---|---|
| **Envoy as ambassador** | Run Envoy as a focused outbound proxy (config-only difference from mesh use). |
| **Cloud SQL Proxy** | GCP-specific ambassador for connecting to managed Postgres/MySQL. |
| **AWS X-Ray daemon** | Sidecar that handles tracing-backend communication. |
| **Vault Agent** | Sidecar for Vault auth; reads secrets and renders them for the app. |
| **Dapr sidecar** | "Distributed Application Runtime" — generalizes ambassador (and other) patterns. |
| **NGINX / HAProxy as outbound proxy** | Simple ambassador implementations for HTTP. |
| **Service-mesh egress** | Istio's `ServiceEntry` + sidecar handles this when mesh is deployed. |
| **Stripe's recommended client wrapping** | Per-language libraries that act as in-process ambassadors. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Google Cloud customers** | Cloud SQL Proxy is the standard for connecting to managed databases — a canonical ambassador. | ✅ Verified — [Cloud SQL Proxy docs](https://cloud.google.com/sql/docs/postgres/sql-proxy) |
| **AWS users** | X-Ray daemon and many other AWS sidecars follow the ambassador pattern. | ✅ Verified — AWS docs |
| **HashiCorp customers** | Vault Agent is widely deployed as an ambassador-style sidecar. | ✅ Verified — HashiCorp docs |
| **Microsoft Dapr users** | Dapr's outbound bindings standardize ambassador-style calls. | ✅ Verified — [Dapr.io](https://dapr.io/) |
| **Stripe-integrated companies** | Recommended setup for high-volume merchants includes ambassador-style local wrappers. | ✅ Verified — Stripe documentation |
| **Many K8s-native shops** | Use Envoy as a focused outbound proxy when full service mesh isn't justified. | ⚠ Common pattern, not always documented |

---

## Further reading

- Microsoft Azure Architecture Center, *Ambassador pattern*.
- Brendan Burns et al., *Design patterns for container-based distributed systems* (HotCloud 2016) — formalized Ambassador alongside Sidecar and Adapter. [PDF](https://www.usenix.org/system/files/conference/hotcloud16/hotcloud16_burns.pdf).
- Brendan Burns, *Designing Distributed Systems* (O'Reilly) — Ch on multi-container patterns including ambassador.
- Dapr.io documentation — generalization of the pattern.
- Cloud SQL Proxy documentation — concrete production example.
- Vault Agent documentation — secret-management ambassador.
- *Service Mesh and Ambassador: When to Use Which* — multiple blog posts comparing scope (Buoyant, Solo.io).

---

*Diagram sources: [`../diagrams/src/ambassador-pattern.d2`](../diagrams/src/ambassador-pattern.d2), [`../diagrams/src/sidecar-pattern.d2`](../diagrams/src/sidecar-pattern.d2).*
