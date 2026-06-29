# Consumer-Driven Contract Test

**Aliases:** CDC Test (not Change Data Capture!), Pact Test, API Contract Test, Consumer-Driven Contract (CDC)
**Category:** Testing / Communication
**Sources:**
[Chris Richardson — microservices.io: Consumer-Driven Contract Test](https://microservices.io/patterns/testing/service-integration-contract-test.html) ·
[Ian Robinson, *Consumer-Driven Contracts: A Service Evolution Pattern* (martinfowler.com, 2006)](https://martinfowler.com/articles/consumerDrivenContracts.html) ·
[Pact documentation](https://docs.pact.io/) ·
Sam Newman, *Building Microservices* (2nd ed.), Ch on Testing

---

## Problem

> [!TIP]
> **ELI5.** In a microservices system, one service calls another (Orders calls Customers). What happens when the Customers team renames a field? Or removes one? Or changes the response shape? The Orders code that consumed that field breaks — silently, until a real request hits the broken path in production. Traditional integration tests catch this only when you spin up *all* services together, which is expensive and slow. **Consumer-Driven Contract Tests** flip the problem: the consumer (Orders) writes down its expectations as a "contract" file; the producer (Customers) runs that file as a test in CI. If Customers makes a change that violates the contract, *their* build breaks — they find out before merging.

In a microservices system, services depend on each other's APIs. The Orders service calls the Customers service via REST or gRPC. The Customers team is free to change their API — that's the point of independent services. But changes can break consumers:

- **Renamed field**: `email` becomes `email_address` → Orders code can't find it.
- **Removed field**: Customers stops returning `phone` → Orders breaks if it used it.
- **Changed type**: `customer_id` was a number, becomes a string → deserialization fails.
- **Changed semantics**: `is_active` used to mean "ever active"; now means "active in last 30 days" → Orders code makes wrong decisions silently.
- **Status code changes**: 404 was thrown for "not found"; now 400 → Orders' error-handling path doesn't trigger.

These breakages aren't caught by:

- **Producer's own tests** — they verify the producer's behavior, but the producer doesn't know which fields each consumer relies on. The change "looks fine" from inside Customers.
- **Consumer's unit tests** — they test the consumer against a mock; the mock returns whatever the consumer's test expected, not what production returns.
- **End-to-end tests** — they would catch it but only when run with the latest both services together, in a fully-integrated environment that's slow, flaky, and run too rarely.

The result: breaking changes ship to production, found by real traffic. Or, more insidiously, services accumulate fear of changing anything — every API field becomes load-bearing whether or not anyone uses it.

**Consumer-Driven Contract Tests** invert the testing model: the *consumer* defines its expectations as a machine-readable contract; the *producer* verifies its API satisfies all consumers' contracts. Breaking changes become impossible to merge silently — the producer's CI fails when any consumer's contract is violated.

## How it works

> [!TIP]
> **ELI5.** The Orders team writes a test that says: "When I send `GET /customers/42` to Customers service, I expect a 200 response with these specific fields and types." This expectation is exported to a small file (the contract). The file is shared via a broker (Pact Broker, or just a Git repo). When the Customers team runs their CI, they pull all consumers' contracts and replay them against the real Customers service. If a contract fails, the build fails — Customers can't ship a change that breaks any consumer without knowing.

The flow:

![Consumer-Driven Contract Test: full lifecycle](../diagrams/svg/cdc-test-flow.svg)

**Step 1 — Consumer expresses expectations.** The Orders team writes a contract test using a tool like Pact, Spring Cloud Contract, or Spec-mate:

```javascript
// In Orders' codebase
pact.given('customer 42 exists')
    .uponReceiving('a get customer request')
    .withRequest({ method: 'GET', path: '/customers/42' })
    .willRespondWith({
      status: 200,
      body: {
        id: like(42),
        name: like('Alice'),
        email: like('alice@example.com'),
        address: { ... }
      }
    });
```

The test generates a **contract file** — typically a JSON document describing the request shape and the expected response. The contract is uploaded to a **contract broker** (Pact Broker, S3, or even a Git repo).

**Step 2 — Consumer verifies its own behavior against a stub.** Pact spins up a local HTTP stub that satisfies the contract. The Orders code runs against the stub; the test verifies Orders does the right thing with that response. If Orders' code correctly handles a 200 with the expected body, the test passes. Orders' CI is now confident: "If Customers honors this contract, my code works."

**Step 3 — Producer verifies it satisfies all contracts.** When the Customers service runs its CI, it pulls all contracts from the broker that mention Customers. For each contract:
1. Set up the precondition state (e.g., "customer 42 exists").
2. Send the request the contract describes.
3. Assert the response matches the contract.

If any contract fails, the producer build fails. Customers cannot ship a change that breaks any documented consumer.

**Step 4 — Change handling.** When Customers wants to make a change (e.g., remove `email`), the contract verification fails because Orders' contract still expects `email`. The Customers team now sees the failure *before* merging — they can:
- Talk to Orders: do they still need `email`?
- Deprecate via versioning: add `/v2` without `email`, leave `/v1` intact.
- Wait for Orders to update their contract first, then make the change.

The key benefit: **no surprise breakages in production**. Every breaking change is visible at producer-build time.

### Why "consumer-driven"?

The pattern is called consumer-driven because **the consumer's needs define the contract**. The producer's API surface might expose 50 fields, but if Orders only uses 5, Orders' contract specifies those 5. If Customers wants to remove the other 45, no contract is violated — they can do it. If they want to remove one of Orders' 5, the contract catches it.

This shifts the conversation: instead of "this is what I return," it becomes "this is what someone is relying on." Producers gain visibility into what's load-bearing.

### Pact: the dominant implementation

**Pact** is the dominant tool for consumer-driven contracts, with implementations in most major languages (Java, JavaScript, Ruby, Go, Python, .NET, Rust, Swift). Pact Broker is the hosted contract repository, with features for:
- Versioning contracts.
- Tracking which versions of consumers and producers are compatible.
- "Can-I-deploy" checks: "can Orders v3.4.0 be deployed safely given current Customers state?"
- Webhooks to trigger downstream pipelines.

Pact has become the de facto standard for HTTP/REST contracts. For other protocols:
- **gRPC**: Pact supports gRPC; alternative is buf's schema breakage detection.
- **Message-based (Kafka)**: Pact supports message contracts; alternative is Schema Registry compatibility rules.
- **GraphQL**: Apollo provides schema-evolution checks that serve a similar role.

### Compared to schema registries

For message-based systems, **schema registries** (Confluent Schema Registry, Apicurio) provide a related capability: schema compatibility checking. Producers register their schema; consumers register theirs; the registry enforces compatibility rules (backward, forward, full) when new versions are registered.

Schema registries catch *schema* breakage (field types, structure). They don't catch *semantic* breakage — fields that change meaning, status codes that shift, sequences of calls that no longer work. Contract tests can catch all of these because they're real test assertions, not just schema validation.

The two are complementary. Schema registries for high-volume Kafka pipelines; contract tests for cross-service API calls.

### Trade-offs

The advantages:

- **Catches breakage before merge** — much earlier than e2e tests would.
- **Fast** — runs in single-service CI; no integrated environment needed.
- **Explicit about coupling** — the contract makes consumer-producer dependencies visible.
- **Forces conversation** — when a contract breaks, the teams must talk.
- **Documentation** — contracts double as machine-checked API documentation.

The disadvantages:

- **Discipline required** — both teams must maintain contracts; if consumers skip writing them, you have no protection.
- **Tooling complexity** — Pact Broker, CI integration, version management add operational overhead.
- **Contracts can drift** — a contract written 6 months ago might not reflect what the consumer actually does today.
- **Hard for stateful interactions** — multi-step flows, async event sequences are harder to contract-test cleanly.
- **Hard with very many consumers** — N consumers × M producers = N × M contracts to manage.

The discipline issue is the biggest in practice. Contract tests are most valuable when they're consistently applied; one team skipping them breaks the model.

### Where it fits in the test suite

Contract tests live alongside other test types:

- **Unit tests** — internal logic, fast, many.
- **Integration tests** — adapter to a real dependency (DB, queue).
- **Contract tests** — verify the API agreement with consumers/producers.
- **Service component tests** — verify the entire service in isolation.
- **End-to-end tests** — verify the whole system; use sparingly.

The contract test fills a specific gap: testing the *agreement* between services without spinning up both. It's the most valuable microservices-specific test type.

### Adoption strategy

Successful adoption usually goes:

1. **Start with one pair** — pick one consumer-producer pair where breakages are painful. Add Pact, set up Pact Broker.
2. **Run for a release cycle.** See real value: catch a real breakage.
3. **Expand to other consumers of the same producer.** Producer now has multiple contracts; one CI run validates all.
4. **Roll out to other producer-consumer pairs.** Each addition is low-friction once Pact Broker is set up.
5. **Make it part of the service template.** New services get Pact integration by default.

Full coverage often takes a year for a large org; the value is incremental.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Pact (HTTP)** | The dominant implementation; cross-language. |
| **Pact (Message)** | Same model for Kafka/queue messages. |
| **Spring Cloud Contract** | Java-centric alternative; uses Groovy/YAML for contracts. |
| **OpenAPI-based testing** | Validate against OpenAPI spec; less consumer-driven, more spec-driven. |
| **Schema Registry** | For Kafka-Avro/Protobuf; structural compatibility. |
| **GraphQL schema checks** | Apollo's schema evolution checking. |
| **buf breakage detection** | gRPC/protobuf schema compatibility. |
| **Diffy / shadow traffic** | Production-traffic alternative; catches real-world breakage in shadow. |

## When NOT to use

- **Single client, single producer** that change together — overhead not worth it.
- **Pure batch jobs** with no API surface.
- **Without organizational commitment** to maintain contracts.
- **When schema registry suffices** (high-volume async messaging with strict schema rules).

---

## Real-world implementations

| Tool | Notes |
|---|---|
| **Pact** | The dominant tool; cross-language; mature. |
| **Pact Broker / PactFlow** | Hosted contract repository. |
| **Spring Cloud Contract** | Java-focused alternative. |
| **Confluent Schema Registry** | For Kafka schemas. |
| **Apicurio Registry** | Alternative schema registry. |
| **buf** | Protobuf/gRPC schema-evolution checking. |
| **OpenAPI Diff** | Structural OpenAPI comparison. |
| **Apollo Schema Checks** | GraphQL schema evolution. |
| **Diffy (Twitter)** | Shadow-traffic comparison tool. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Atlassian** | Public engineering posts on internal Pact adoption. | ✅ Verified — Atlassian Engineering blog |
| **REA Group** | Originated much of the Pact ecosystem; published case studies. | ✅ Verified — REA tech blog and Pact contributors |
| **DAZN, IAG, BBC, GOV.UK** | Public adoption stories. | ✅ Verified — Pact case studies page |
| **Spotify** | Public engineering on consumer-driven contracts for internal APIs. | ⚠ Mentioned in talks; specific architecture varies |
| **PactFlow customers (many enterprises)** | Pact Broker SaaS customers across industries. | ✅ Verified — PactFlow case studies |

---

## Further reading

- Ian Robinson, *Consumer-Driven Contracts: A Service Evolution Pattern* (2006) — the foundational essay. [martinfowler.com/articles/consumerDrivenContracts.html](https://martinfowler.com/articles/consumerDrivenContracts.html).
- Pact documentation — the most comprehensive practical guide.
- Chris Richardson, *microservices.io* — Consumer-Driven Contract Test pattern.
- Sam Newman, *Building Microservices* (2nd ed.), Ch on Testing.
- *Building Evolutionary Architectures* (Ford, Parsons, Kua) — contracts in the context of evolution-friendly design.
- Confluent blog — Schema Registry and contract testing.
- Adam Hawkins's podcast / blog on Pact in production.

---

*Diagram sources: [`../diagrams/src/cdc-test-flow.d2`](../diagrams/src/cdc-test-flow.d2), [`../diagrams/src/test-pyramid.d2`](../diagrams/src/test-pyramid.d2).*
