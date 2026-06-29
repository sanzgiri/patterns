# Chaos Engineering

**Aliases:** Chaos Engineering, Fault Injection Testing, Failure Testing, Resilience Engineering, Game Days
**Category:** Operations / Resilience
**Sources:**
[principlesofchaos.org](https://principlesofchaos.org/) — *The Principles of Chaos Engineering* ·
[Netflix Tech Blog — *The Netflix Simian Army* (2011)](https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116) ·
[Netflix Tech Blog — *Chaos Engineering Upgraded* (2015)](https://netflixtechblog.com/chaos-engineering-upgraded-878d341f15fa) ·
*Chaos Engineering* (Casey Rosenthal, Nora Jones, O'Reilly 2020) ·
[Google SRE Book — DiRT (Disaster Recovery Testing)](https://sre.google/sre-book/availability-table/) ·
[Microsoft Azure — Chaos Studio docs](https://learn.microsoft.com/en-us/azure/chaos-studio/) ·
[AWS Fault Injection Service docs](https://docs.aws.amazon.com/fis/) ·
[Gremlin documentation](https://www.gremlin.com/docs/) ·
[Chaos Mesh, LitmusChaos (CNCF) documentation](https://chaos-mesh.org/)

---

## Problem

> [!TIP]
> **ELI5.** You've built a "resilient" distributed system: circuit breakers, retries, fallbacks, multi-AZ deployments, failover. You think it works. You hope it works. But you don't actually know — until something fails for real, usually at 3am, in a way you didn't plan for. **Chaos engineering** is the discipline of deliberately breaking parts of your system in controlled ways, *while watching what happens*, to find weaknesses before users do. It started at Netflix with "Chaos Monkey" — a program that randomly killed production instances during business hours so engineers were forced to build services that survive instance loss. The discipline has grown from "kill a server" to a structured experimentation practice: hypothesis → small perturbation → measurement → learning.

The motivating insight, from Netflix circa 2010: as systems grew bigger and more distributed, *unforeseen failures* dominated. The team would build something they thought was resilient — and find out it wasn't only when an AWS instance died at 4am and took down a critical path that nobody had thought to test.

The classical alternative — disaster-recovery testing in a staging environment — has fatal weaknesses:

- Staging doesn't have production's scale, traffic patterns, or dependencies.
- Tests run on schedule (monthly DR drill), not against the system's real day-to-day state.
- Many failures are emergent — only appear under specific load, specific configurations, specific upstream behaviors that can't be reproduced in staging.
- A passing DR test doesn't mean it'll pass when the same dep fails differently next time.

Netflix's radical answer: **inject failure continuously in production**. If killing a random instance always breaks something, that's a bug; fix it. If it never breaks anything, you've built true resilience.

This was contrarian in 2010. By 2020 it's mainstream — AWS, Azure, GCP all sell chaos-engineering services; Gremlin built a chaos company; principlesofchaos.org formalized the practice; CNCF has multiple chaos projects (Chaos Mesh, LitmusChaos). The cultural shift is the bigger story: "failure is normal, plan for it, exercise it" instead of "failure is exceptional, avoid it."

## How it works

> [!TIP]
> **ELI5.** Pick a real failure mode (kill an instance, slow a network link, fill a disk, crash a dependency). Make a prediction about what should happen (steady-state hypothesis). Inject the failure on a small portion of production with a tight blast-radius limit and an abort plan. Watch your dashboards. If the system stays within hypothesis, your confidence in resilience grows. If it doesn't, you found a weakness; fix it and re-run. Repeat with bigger experiments as confidence grows.

### The experiment loop

The structured methodology, from principlesofchaos.org:

![Chaos engineering experiment loop](../diagrams/svg/chaos-engineering.svg)

**1. Define the steady-state hypothesis.** What does "healthy" look like, measurably? Examples:
- p99 latency stays below 500ms.
- Error rate stays below 0.1%.
- Checkout success rate stays above 99.5%.
- Cart abandonment rate stays in normal range.

The hypothesis is a measurable, **user-facing** outcome — not "all servers are up." A system can have servers down and users still happy (resilience working) or all servers up and users unhappy (regression unrelated to chaos).

**2. Choose a real-world event.** Real-world failures, not abstract ones:
- Instance termination (the original Chaos Monkey).
- Network latency injection (200ms added to payment API).
- Network partition between AZs.
- DNS resolution failure.
- Disk fill on a critical host.
- CPU saturation.
- Memory pressure.
- Process kill (kernel signals).
- Clock skew.
- Dependency service slow/erroring.

The model is *real failures that have happened or will happen*, not arbitrary computational disruptions.

**3. Contain blast radius.** Critical safety practice:
- Start in dev/staging if practice is new.
- Choose a known time window (business hours so engineers are awake).
- Limit the experiment to a small fraction of traffic (1% of users, 1 instance out of 1000).
- Define abort criteria upfront: "if error rate exceeds 0.5%, stop immediately."
- Have an automated abort: most chaos tools include a "magic switch" that aborts when monitors degrade.

The rule: every experiment must be safer than the production failure it simulates. If chaos breaks customers, you have done it wrong.

**4. Run the experiment.** Inject the fault, monitor the steady-state. The team is on standby; metrics are watched intently for the duration. Abort if hypothesis breaks.

**5. Analyze and share.** Did the hypothesis hold? If yes — confidence increases; document the experiment's evidence; expand scope next time. If no — you found a real weakness. File bugs, fix the underlying resilience gap, re-run after fix. Share findings widely so other teams learn.

The discipline is the experiment loop, not the destruction. **Chaos without hypothesis is just sabotage.**

### Netflix's Simian Army (the OG catalog)

Netflix's chaos toolkit started as Chaos Monkey (2010) and grew into a "Simian Army" of specific failure simulators:

![Netflix Simian Army](../diagrams/svg/simian-army.svg)

- **Chaos Monkey** — terminates random instances. The OG. Forces services to handle instance loss gracefully.
- **Chaos Gorilla** — simulates entire **Availability Zone** failure. Forces multi-AZ failover.
- **Chaos Kong** — simulates entire **AWS region** failure. The most ambitious; verifies global failover for the most catastrophic scenarios. Netflix successfully ran Chaos Kong as a regular practice.
- **Latency Monkey** — injects latency into RPC calls. Tests timeout and circuit-breaker handling.
- **Conformity Monkey** — finds instances that don't match conventions (missing tags, wrong AMI). Operational hygiene, not chaos per se.
- **Doctor Monkey** — health-check inspection.
- **Janitor Monkey** — finds and removes unused resources (cost tool).
- **Security Monkey** — finds security misconfigurations.

Most have been retired in favor of the unified **ChAP (Chaos Automation Platform)**, which runs continuous, automated, hypothesis-driven experiments. But the catalog formalized "what kinds of things should you inject?" and remains influential.

### Beyond Netflix

By 2020, chaos engineering had become mainstream:

| Org / Tool | Notes |
|---|---|
| **AWS Fault Injection Service (FIS)** | Managed fault injection for AWS resources. |
| **Azure Chaos Studio** | Microsoft's managed offering. |
| **GCP Chaos Toolkit / Lithos** | Various GCP integrations. |
| **Gremlin** | Commercial chaos-as-a-service; pioneered the category. |
| **Chaos Mesh** | CNCF; Kubernetes-native chaos. |
| **LitmusChaos** | CNCF; Kubernetes-native chaos. |
| **Pumba** | Docker container chaos. |
| **Toxiproxy** | Shopify's; network-level chaos. |
| **kube-monkey** | K8s pod killer. |
| **Powerful Seal** | Bloomberg's K8s chaos tool. |
| **Mangle** | VMware's. |
| **Chaos Toolkit** | OSS framework / DSL. |

### Game Days

A complementary practice (Amazon called these "Game Days" before chaos engineering was named): scheduled exercises where the team simulates an incident — a region outage, a security breach, a dependency failure — and the on-call team responds as if it's real. Distinct from automated chaos because:

- **Scheduled and announced**, not surprise.
- **Tests human/process response**, not just system response.
- **Involves runbooks, incident management, communication channels**.
- **Often broader scope** (executive notification, customer comms simulation).

A mature program has both: continuous automated chaos (system resilience) + periodic game days (organizational resilience).

### Where to inject — the dependency map

Different injection points test different things:

- **Instance / pod**: tests deployment redundancy.
- **Process kill (SIGKILL)**: tests graceful shutdown and restart handling.
- **CPU/memory pressure**: tests resource isolation, bulkheads.
- **Disk fill / IO degradation**: tests log/cache eviction, fail-safe modes.
- **Network latency**: tests timeouts, retries, circuit breakers.
- **Network packet loss**: tests TCP retransmit, idempotency.
- **Network partition**: tests split-brain handling, CAP trade-offs.
- **DNS failure**: tests DNS caching, fallback resolution.
- **Dependency 5xx**: tests fallback paths.
- **Dependency slow response**: tests timeout configuration.
- **Time skew (clock jump)**: tests time-sensitive logic.
- **Region failure**: tests multi-region failover.

A mature program has experiments at each level — each tests different resilience properties.

### When you're "ready" for chaos

A frequent question: is my system ready for chaos engineering? The general guidance:

**Prerequisites that should be in place first**:
- Strong **observability** — you need to see what's happening (logs, metrics, tracing all in place).
- Strong **alerting** — abort signals must fire reliably.
- A **rollback mechanism** — for both code and the chaos experiment itself.
- **Cell / AZ / region isolation** of some kind — so a chaos experiment can't take everything down.
- **Incident response practices** — even with care, chaos experiments occasionally trigger real incidents.
- **Cultural buy-in** — leadership accepts that chaos is valuable even when it causes brief disruption.

Without observability, chaos is just damage. Without rollback, it's permanent damage. Without buy-in, the first incident kills the program.

Start small: kill a pod in dev. Then in staging. Then in production for low-traffic services. Then increase the failure-mode catalog. Then for high-traffic services with tight blast-radius controls.

### Anti-patterns

- **Chaos without hypothesis**: random destruction without prediction is sabotage, not engineering.
- **Chaos without abort criteria**: blowing up production at 3am is a real bad day.
- **Chaos in fragile systems**: don't add chaos before you've built the basics; you'll just have outages.
- **Chaos to test "should we have multi-AZ"**: you should already know the answer; chaos verifies what you've designed.
- **Chaos without sharing findings**: failures discovered must be communicated, fixed, and re-tested.
- **One-time chaos**: chaos must be continuous because systems change continuously.
- **Confusing chaos with QA/testing**: chaos validates resilience in *production*, not function correctness in dev.

### Where chaos doesn't fit (yet)

- **Single-instance systems**: nothing to fail over to.
- **Heavily-stateful systems with no redundancy**: chaos = data loss.
- **Hard real-time / safety-critical**: medical, aviation, automotive — chaos requires environments that can't be in production.
- **Heavily-regulated changes**: chaos experiments need regulatory acceptance in some industries.
- **Insufficient observability**: chaos without ability to detect impact is dangerous.

For these, simulation environments, fault-injection in test, and Game Days substitute.

### What you get from a mature chaos program

- **Verified resilience**: not "we think it works" but "we've watched it work 100 times."
- **Discovered weaknesses**: ones you'd never have found in design review.
- **Better on-call**: engineers are familiar with failure modes from chaos exposure.
- **Better dependency tracking**: chaos forces you to know what depends on what.
- **Calibrated SLOs**: real evidence of how the system behaves under stress.
- **Faster incident response**: trained reflexes from chaos events.
- **Cultural shift**: "failure is normal" replaces "failure is exceptional."

The Netflix observation: after years of chaos engineering, **outages became boring** — well-understood, well-handled, frequently survived without user impact. The transformation is real, not marketing.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Chaos Monkey-style** | Random instance termination. |
| **Latency injection** | Network slowness. |
| **Network partition** | Split-brain simulation. |
| **Resource pressure** | CPU/memory/disk exhaustion. |
| **Dependency failure simulation** | Mock dependency failures (5xx, slow). |
| **Region / AZ failure (Chaos Kong/Gorilla)** | Large-scale failure. |
| **Game Days** | Scheduled human-and-system exercise. |
| **Continuous chaos / automated** | Run experiments on schedule. |
| **DiRT (Google)** | Annual Disaster Recovery Testing. |
| **Failure-driven development** | Design with failure modes upfront. |
| **[Bulkhead](../res/bulkhead.md)** | Pattern chaos validates. |
| **[Circuit Breaker](../res/circuit-breaker.md)** | Pattern chaos validates. |
| **[Health Endpoint](log-aggregation-metrics.md)** | Used by chaos for abort signals. |
| **[Deployment Stamps](deployment-stamps.md)** | Isolation level for chaos. |

## When NOT to use

- **Without observability** — you won't see what's happening.
- **Without abort capability** — you can't stop bad experiments.
- **In fragile, single-point-of-failure systems** — fix the basics first.
- **In safety-critical real-time systems** — staging environments instead.
- **Without leadership buy-in** — first incident will kill the program.
- **As a substitute for design** — chaos validates design; doesn't replace it.

---

## Real-world implementations

| Tool | Notes |
|---|---|
| **AWS Fault Injection Service (FIS)** | Managed AWS chaos. |
| **Azure Chaos Studio** | Managed Azure chaos. |
| **Gremlin** | Commercial chaos platform. |
| **Chaos Monkey / Simian Army / ChAP** | Netflix's tools (some OSS). |
| **Chaos Mesh** | CNCF; K8s. |
| **LitmusChaos** | CNCF; K8s. |
| **Chaos Toolkit** | OSS framework. |
| **Toxiproxy** | Shopify; network chaos. |
| **Pumba** | Docker chaos. |
| **kube-monkey** | K8s pod killer. |
| **Steadybit** | Commercial. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Netflix** | Originated the practice; published toolset and methodology. | ✅ Verified — [Netflix Tech Blog](https://netflixtechblog.com/) |
| **Google** | DiRT (Disaster Recovery Testing) since 2006. | ✅ Verified — Google SRE book |
| **Amazon / AWS** | Game Days throughout; AWS FIS is a productization. | ✅ Verified — AWS blog |
| **Microsoft Azure** | Chaos Studio offering; internal practices. | ✅ Verified — Azure docs |
| **Facebook / Meta** | Storm / Cyclone for testing data center disaster scenarios. | ✅ Verified — Engineering talks |
| **LinkedIn** | Waterbear; ongoing chaos program. | ✅ Verified — LinkedIn Engineering blog |
| **Slack** | Disasterpiece Theater (game days). | ✅ Verified — Slack Engineering blog |
| **Uber** | uMonkey, Hailstorm; chaos at scale. | ✅ Verified — Uber Engineering blog |
| **Capital One** | Public adopter; financial-services chaos. | ✅ Verified — Capital One tech blog |
| **Many large orgs** | Some level of chaos is now standard. | ✅ Industry-wide adoption |

---

## Further reading

- principlesofchaos.org — the canonical principles.
- *Chaos Engineering* (Rosenthal, Jones — O'Reilly 2020) — the textbook.
- *Learning Chaos Engineering* (Miles — O'Reilly).
- Netflix Tech Blog series on chaos engineering and Chaos Kong.
- *Site Reliability Engineering* (Google SRE book) — DiRT chapter.
- *The DevOps Handbook* — chapter on injecting failure.
- *Drift into Failure* (Sidney Dekker) — broader resilience-engineering perspective.
- Casey Rosenthal's keynote talks — historical perspective from the Netflix chaos team.
- *Reliable Software via SLOs* — defines what chaos is preserving.

---

*Diagram sources: [`../diagrams/src/chaos-engineering.d2`](../diagrams/src/chaos-engineering.d2), [`../diagrams/src/simian-army.d2`](../diagrams/src/simian-army.d2).*
