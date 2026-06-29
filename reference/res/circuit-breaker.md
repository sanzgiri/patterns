# Circuit Breaker

**Aliases:** —
**Category:** Resilience
**Sources:**
[Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) ·
[microservices.io](https://microservices.io/patterns/reliability/circuit-breaker.html) ·
[ByteByteGo](https://github.com/ByteByteGoHq/system-design-101) ·
Nygard, *Release It!* (2007) — original

---

## Problem

> [!TIP]
> **ELI5.** When the friend you keep calling stops picking up, calling them again and again wastes your battery *and* doesn't help them charge their phone. Worse, while you're stuck on each call waiting for them to pick up, you can't call anyone else.

In a distributed system, a service routinely calls others over the network. When a downstream dependency becomes slow or unhealthy, naive clients keep calling it. Three bad things happen:

1. **Thread/connection exhaustion** — calls pile up waiting on timeouts, exhausting the caller's resources.
2. **Cascading failure** — the caller becomes unhealthy too, and its callers in turn become unhealthy.
3. **No recovery room** — the struggling dependency keeps getting hammered and can never catch up.

Retries (without backoff) and unbounded timeouts make all of this worse.

## How it works

> [!TIP]
> **ELI5.** After your friend misses a few calls in a row, stop trying for a while. Every so often, try *one* test call. If they pick up, go back to calling normally. If not, wait even longer before the next test call.

The pattern interposes a **Breaker** object between the application and every remote dependency. The breaker is small but stateful: it remembers recent outcomes (success / failure / timeout) and decides, on each call, whether to let it through or to fail it immediately.

A typical request travels through four components:

![Circuit Breaker Request Flow](../diagrams/svg/circuit-breaker-flow.svg)

The **Client** sends a request to the **Application**. Before issuing the outbound network call, the application consults the **Breaker**, which holds the current state, a failure counter, and the timestamp of the last failure (the cylinder in the diagram). If the breaker permits the call, it forwards to the **Downstream Service** (green path) and updates its counters when the response — success, error, or timeout — comes back. If the breaker refuses, it short-circuits to a **Fallback** path (red dashed line): a cached value, a default response, a queued retry, or simply a fast error. Either way, the application returns a result to the client. Two things matter in this picture:

- The breaker must distinguish **fast failures** (connection refused, 5xx) from **slow failures** (timeouts). Slow failures are more dangerous and should count more heavily — which is why production breakers always enforce a **call timeout** as their first line of defense.
- Failures are tracked **per dependency**, not globally. One breaker per downstream service. A single shared breaker would mean trouble in one dependency trips calls to unrelated ones — creating a different kind of cascading failure.

The interesting behavior lives in the breaker's state machine, which has three states:

![Circuit Breaker State Machine](../diagrams/svg/circuit-breaker-states.svg)

**CLOSED** is the healthy steady state. Every request is allowed through, and the breaker counts failures — either consecutively or as a rolling-window error rate. Once the count crosses `failure_threshold` (commonly 5 failures in 10 seconds, or 50% errors over the last 100 calls), the breaker **trips OPEN**.

In **OPEN**, every incoming call fails immediately without touching the downstream service. This protects the caller (no wasted threads waiting on timeouts) and the callee (no extra load while it recovers). A cooldown timer starts — typically 30 to 60 seconds.

When the cooldown elapses, the breaker transitions to **HALF-OPEN**. The very next request is allowed through as a *trial*. If that trial **succeeds**, the breaker returns to CLOSED, the failure counters reset, and normal operation resumes. If the trial **fails**, the breaker snaps straight back to OPEN and the cooldown restarts — usually with exponential backoff, so a dependency that keeps failing gets probed less and less often.

HALF-OPEN is the design's central trick: it lets the system *probe* recovery cheaply (one request) instead of either flooding the dependency the moment the timer expires or waiting forever for a human to flip a switch.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Count-based breaker** | Trips after N consecutive failures. Simple; sensitive to bursts. |
| **Rate-based / rolling-window breaker** | Trips on % failures over a window. Smoother; what Resilience4j and Polly default to. |
| **Bulkhead + Breaker** | Bulkhead isolates the *resource pool* per dependency so one slow dependency can't starve others; breaker decides when to give up on that pool. They compose. |
| **Retry + Breaker** | Retry handles *transient* failures; breaker handles *sustained* failures. Retry inside breaker, never the other way round. |
| **Throttling** | Limits *callers* (server-side). Breaker protects against *callees* (client-side). Same goal — system stability — from opposite ends. |

## When NOT to use

- **In-process calls** — pure overhead; let them throw.
- **Calls that are already idempotent, cheap, and bounded-latency** (in-region cache reads). The breaker's bookkeeping cost can outweigh the harm of an occasional failure.
- **When you don't have a meaningful fallback.** A breaker that trips to "return 500 immediately" is sometimes still worth it (you free threads), but if the only behavior is "fail faster," better timeouts may serve you better.

---

## Real-world implementations (open source / standard)

| Library | Runtime | Notes |
|---|---|---|
| **Netflix Hystrix** | JVM | Popularized the pattern at scale. Put in maintenance mode in 2018; Netflix recommends Resilience4j for new work. ([source](https://github.com/Netflix/Hystrix/wiki)) |
| **Resilience4j** | JVM | Hystrix's de-facto successor; lighter, functional API; Spring Boot integration. |
| **Polly** | .NET | Built-in support for breaker, retry, timeout, bulkhead, fallback. Used by ASP.NET Core. |
| **opossum** | Node.js | Most-used Node circuit-breaker library. |
| **gobreaker** | Go | Standard breaker library used by many Go services. |
| **Envoy Proxy** | data plane | Outlier detection + connection limits implement breaker behavior at the L7 proxy layer — used by Istio, Consul Connect, AWS App Mesh. ([docs](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier)) |
| **AWS SDKs** | many | Adaptive retry mode uses a token-bucket-based breaker concept to avoid retry storms. |

## Companies using it (verification status marked)

| Company | Use | Status |
|---|---|---|
| **Netflix** | Originated Hystrix to protect against cascading failures in its microservices fleet (notably the 2011 outage that motivated the project). | ✅ Verified — [Netflix Tech Blog, *Fault Tolerance in a High Volume, Distributed System*, 2012](https://netflixtechblog.com/fault-tolerance-in-a-high-volume-distributed-system-91ab4faae74a) |
| **Amazon (AWS)** | Adaptive retry / breaker behavior built into AWS SDKs and described in the AWS Builders' Library. | ✅ Verified — [Builders' Library: *Timeouts, retries, and backoff with jitter*](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) |
| **Google (Envoy at large)** | Envoy outlier detection underpins Istio service mesh used by many Google products. | ✅ Verified at the Envoy level; specific product use is internal |
| **Uber** | Has discussed breaker + bulkhead use in their service mesh. | ⚠ Well-known via conference talks; specific public link not re-verified |
| **Stripe** | Uses idempotency + retries + breakers in API gateway layer. | ⚠ Mentioned in talks; not re-verified by fetch |
| **Microsoft Azure** | Recommends and documents the pattern as a first-class cloud design pattern. | ✅ Verified — [pattern doc](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) |

**⚠ marks claims that are widely-known industry attribution but were not re-verified by primary-source fetch for this document. Treat as plausible but not citation-grade.**

---

## Further reading

- Michael Nygard, *Release It! Design and Deploy Production-Ready Software*, 2nd ed. — chapter on Stability Patterns introduced this and Bulkhead in 2007.
- Martin Fowler, *CircuitBreaker* — [martinfowler.com/bliki/CircuitBreaker.html](https://martinfowler.com/bliki/CircuitBreaker.html)
- Resilience4j docs — [resilience4j.readme.io](https://resilience4j.readme.io/docs/circuitbreaker)
- AWS Builders' Library — *Timeouts, retries, and backoff with jitter* (related and complementary)

---

*Diagram sources: [`../diagrams/src/circuit-breaker-flow.d2`](../diagrams/src/circuit-breaker-flow.d2), [`../diagrams/src/circuit-breaker-states.d2`](../diagrams/src/circuit-breaker-states.d2). Render with `d2 input.d2 output.svg`.*
