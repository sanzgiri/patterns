# System Design and LLM Patterns Curriculum

This curriculum uses the repository as a pattern language rather than as a checklist. The aim is to build the judgment to recognize a problem, select the smallest pattern that solves it, explain the trade-offs, and extend the design when reality breaks the happy path.

It has two outcomes:

1. Build and operate reliable systems that use distributed-system and LLM patterns appropriately.
2. Give clear, structured answers to system-design interviews and LLM/agent case studies.

## How to study

Use a three-pass loop for every pattern:

1. **Mechanism:** draw the components, state transitions, failure modes, and guarantees.
2. **Application:** apply it to a company problem with explicit scale, latency, consistency, cost, and safety requirements.
3. **Transfer:** compare it with two alternatives, then modify it for a changed requirement.

For each session, produce one page containing:

- the problem and forces;
- the invariant or guarantee the pattern preserves;
- a diagram and request/data flow;
- normal path, failure path, recovery path;
- operational signals and test strategy;
- alternatives and when not to use it;
- a short interview answer using requirements → architecture → deep dive → trade-offs.

The default pace is 16 weeks, four study sessions per week, 60–90 minutes each. A faster pace can combine adjacent weeks; the ordering matters more than the calendar.

## Complete 21-week map

| Week | Focus | Company problem | Primary output |
|---|---|---|---|
| 1 | Requirements, boundaries, scaling shape | Global photo sharing | Capacity model and first architecture |
| 2 | Storage, indexes, caches, sketches | E-commerce search and analytics | Read/write model and approximation choices |
| 3 | Sharding, replication, consistency | Global inventory and orders | Consistency contract and failure analysis |
| 4 | Coordination and time | Payment-cluster leadership | Election, lease, and fencing design |
| 5 | Messaging and backpressure | Video processing pipeline | Delivery, retry, and overload design |
| 6 | Distributed writes and workflows | Marketplace checkout | Saga, outbox, CDC, and reconciliation |
| 7 | Resilience and progressive delivery | Ride dispatch outage | Failure-containment and rollout plan |
| 8 | Observability, identity, migration | Legacy bank modernization | Migration architecture and audit model |
| 9 | Augmented LLMs and interfaces | Insurance claims intake | Typed, validated human-in-the-loop assistant |
| 10 | Retrieval and freshness | Enterprise policy assistant | Retrieval and citation architecture |
| 11 | Context engineering and memory | Persistent research assistant | Context/memory lifecycle design |
| 12 | Workflows versus agents | Travel-support automation | Workflow/agent boundary decision |
| 13 | Agent architecture and orchestration | Software-engineering agent | Planner/executor/tool architecture |
| 14 | Agent engineering and evaluation | Customer-ticket resolution agent | Eval harness and production telemetry |
| 15 | Security and containment | Accounts-payable agent | Threat model and permission boundaries |
| 16 | Model choice, rollout, synthesis | Healthcare operations copilot | Capstone design and change adaptations |
| 17 | Platform and service communication | Internal platform for 200 services | Discovery, mesh, configuration, health model |
| 18 | Cross-service APIs and testing | Travel-booking API | Composition and compatibility strategy |
| 19 | Advanced coordination and replicated state | Collaborative document system | Convergence and anti-entropy design |
| 20 | Specialized scale, storage, identity | Global collaboration product | Architecture selection under extreme constraints |
| 21 | LLM/agent extensions | Coding-and-operations agent | Skills, code mode, browser, swarm, eval design |

Every week follows the same four-session rhythm: learn the mechanism; derive it from requirements; design the case; defend the design in an interview. The primary output is reviewed against the mastery criteria at the end of this document.

## Coverage boundary

This is a guided spine, not a page-by-page traversal of the catalog. The system-design reference has 85 pattern pages; 57 are assigned explicitly below, while 28 are either represented by a neighboring pattern or reserved for a later breadth pass. That was a sequencing decision, not a judgment that those patterns are unimportant. In particular, the curriculum should not be described as covering every pattern in 16 weeks.

The currently unassigned system-design pages are:

- architecture/platform: [Compute Resource Consolidation](../reference/arch/compute-resource-consolidation.md), [Microservice Chassis](../reference/arch/microservice-chassis.md), [Service Template](../reference/arch/service-template.md), [Space-Based Architecture and Edge Computing](../reference/arch/space-based-edge.md);
- coordination/building blocks: [Clock-Bound Wait](../reference/block/clock-bound-wait.md), [High-Water Mark and Low-Water Mark](../reference/block/hwm-lwm.md), [Singular Update Queue](../reference/block/singular-update-queue.md), [Consistent Core](../reference/coord/consistent-core.md), [Emergent Leader](../reference/coord/emergent-leader.md);
- data: [CRDT](../reference/data/crdt.md), [Merkle Tree](../reference/data/merkle-tree.md), [Polyglot Persistence](../reference/data/polyglot-persistence.md), [Segmented Log](../reference/data/segmented-log.md);
- communication/platform: [Cache-Managed Strategies](../reference/cache/cache-managed-strategies.md), [Ambassador](../reference/comm/ambassador.md), [API Composition](../reference/comm/api-composition.md), [BFF](../reference/comm/bff.md), [Client-Side Discovery](../reference/comm/client-side-discovery.md), [Consumer-Driven Contract Test](../reference/comm/consumer-driven-contract-test.md), [GraphQL Federation](../reference/comm/graphql-federation.md), [Server-Side Discovery](../reference/comm/server-side-discovery.md), [Service Component Test](../reference/comm/service-component-test.md), [Service Mesh](../reference/comm/service-mesh.md), [Service Registry](../reference/comm/service-registry.md), [Sidecar](../reference/comm/sidecar.md), [State Watch](../reference/comm/state-watch.md);
- operations/identity: [Externalized Configuration](../reference/ops/externalized-config.md), [SCIM, Passkeys, and Kerberos](../reference/sec/scim-passkeys-kerberos.md).

The LLM corpus has 60 pages; 52 are assigned explicitly and 8 are reserved for extension modules. Its omitted pages should not be silently counted as mastered.

## How to incorporate the complete catalog

Use the 16 weeks above as the **depth track**, then add a five-week breadth pass. This preserves the causal learning order while giving every page an explicit place:

### Week 17: Platform and service communication

Cover [Service Template](../reference/arch/service-template.md), [Microservice Chassis](../reference/arch/microservice-chassis.md), [Sidecar](../reference/comm/sidecar.md), [Service Mesh](../reference/comm/service-mesh.md), [Service Registry](../reference/comm/service-registry.md), [Client-Side Discovery](../reference/comm/client-side-discovery.md), [Server-Side Discovery](../reference/comm/server-side-discovery.md), [Ambassador](../reference/comm/ambassador.md), [State Watch](../reference/comm/state-watch.md), and [Externalized Configuration](../reference/ops/externalized-config.md).

Case: build the internal platform for 200 microservices. Compare library, sidecar, and mesh responsibilities; then design service registration, configuration rollout, health watching, and failure behavior.

### Week 18: Cross-service APIs and testing

Cover [API Composition](../reference/comm/api-composition.md), [BFF](../reference/comm/bff.md), [GraphQL Federation](../reference/comm/graphql-federation.md), [Consumer-Driven Contract Testing](../reference/comm/consumer-driven-contract-test.md), and [Service Component Testing](../reference/comm/service-component-test.md).

Case: build a travel-booking API consumed by web, mobile, and partner clients. Decide where composition lives, how schemas evolve, and how to test independently deployed services.

### Week 19: Advanced coordination and replicated state

Cover [Segmented Log](../reference/data/segmented-log.md), [CRDT](../reference/data/crdt.md), [Merkle Tree](../reference/data/merkle-tree.md), [High-Water/Low-Water Mark](../reference/block/hwm-lwm.md), [Clock-Bound Wait](../reference/block/clock-bound-wait.md), [Singular Update Queue](../reference/block/singular-update-queue.md), [Consistent Core](../reference/coord/consistent-core.md), and [Emergent Leader](../reference/coord/emergent-leader.md).

Case: design collaborative editing and anti-entropy repair for a multi-region document system. Identify which guarantees come from ordering, which come from convergence, and which require a single authority.

### Week 20: Specialized scale, storage, and identity

Cover [Cache-Managed Strategies](../reference/cache/cache-managed-strategies.md), [Polyglot Persistence](../reference/data/polyglot-persistence.md), [Space-Based Architecture and Edge Computing](../reference/arch/space-based-edge.md), [Compute Resource Consolidation](../reference/arch/compute-resource-consolidation.md), and [SCIM, Passkeys, and Kerberos](../reference/sec/scim-passkeys-kerberos.md).

Case: design a global enterprise collaboration product with hot working state, regional execution, multiple storage workloads, workforce provisioning, and strong authentication.

### Week 21: LLM and agent extensions

Cover [Coding Agents](../reference-llm/agt/coding-agents.md), [Computer Use](../reference-llm/agt/computer-use.md), [Self-Directed Swarms](../reference-llm/agt/self-directed-swarms.md), [Context Plumbing](../reference-llm/ctx/context-plumbing-reinforcement.md), [Eval Saturation](../reference-llm/qua/eval-saturation.md), [AGENTS.md](../reference-llm/skills/agents-md.md), [Code Mode](../reference-llm/skills/code-mode.md), and [SKILL.md](../reference-llm/skills/skill-md-format.md).

Case: build a coding-and-operations agent for a large engineering organization. Combine repository instructions, skills, code execution, browser use, sub-agents, context reinforcement, and evals that remain useful after the benchmark saturates.

After Week 21, every reference page has an explicit location. Revisit the highest-value patterns through the interview case-study loop; breadth exposure is not the same as mastery.

## The mental model

Most designs in this repository can be understood as a sequence of questions:

1. **Where is truth?** A database, log, event stream, external system, or model output?
2. **Who owns a decision or piece of state?** A service, leader, partition, workflow, or agent?
3. **What happens when messages, machines, clocks, dependencies, or model outputs fail?**
4. **What can be made cheaper by precomputation?** A cache, materialized view, index, embedding, summary, or prompt cache?
5. **How do we observe, evaluate, roll out, and contain change?**

The central intuition is that patterns are mechanisms for preserving invariants under particular forces. Never name a pattern before naming the force.

## Phase I — Distributed-systems foundations

### Week 1: Requirements, boundaries, and scaling shape

Read: [Monolith](../reference/arch/monolith.md), [Service Decomposition](../reference/arch/service-decomposition.md), [Microservices](../reference/arch/microservices.md), [CDN](../reference/scale/cdn.md), [Deployment Stamps](../reference/ops/deployment-stamps.md).

Case: design a global photo-sharing service. Start with a monolith, then identify the first real bottleneck before splitting anything. Estimate traffic, storage, hot keys, read/write ratio, durability, and regional needs.

Interview drill: “Design a URL shortener” and “Design a photo-sharing feed.” The goal is not breadth; it is learning to state assumptions and choose a boundary.

Lesson pack: [Week 1 — Requirements, boundaries, and scaling shape](lessons/week-01.md).

### Week 2: Storage engines, indexes, and caching

Read: [B-Tree](../reference/data/btree.md), [LSM-Tree](../reference/data/lsm-tree.md), [WAL](../reference/data/wal.md), [MVCC](../reference/data/mvcc.md), [Probabilistic Sketches](../reference/data/probabilistic-sketches.md), [Cache-Aside](../reference/cache/cache-aside.md), [Cache Hierarchy](../reference/cache/cache-hierarchy-storage-tiering.md), [Materialized View](../reference/data/materialized-view-index.md), [Inverted Index](../reference/data/inverted-index.md).

Case: design product search and a product-detail page for an e-commerce company. Decide what is authoritative, what is derived, and what is cached. Explain stale reads, invalidation, compaction, write amplification, and rebuilding a view. Add a streaming analytics component: use a Bloom filter to reject definite non-membership cheaply, HyperLogLog to estimate unique visitors, and Count-Min Sketch to find heavy hitters without storing every key.

Extension: **MinHash and locality-sensitive similarity.** MinHash estimates Jaccard similarity between sets using compact signatures. Place it after inverted indexes and sketches: use it for near-duplicate product descriptions, web pages, or support tickets, then compare it with exact hashing, SimHash, embeddings, and ANN search. The key distinction is membership versus cardinality versus frequency versus similarity:

| Need | Appropriate structure | Error shape |
|---|---|---|
| “Could this key exist?” | Bloom filter | False positives, no false negatives |
| “How many distinct keys?” | HyperLogLog | Approximate cardinality |
| “How often does key X occur?” | Count-Min Sketch | Overestimates frequency |
| “How similar are these sets?” | MinHash | Approximate Jaccard similarity |

Interview drill: “Design a news feed” with a requirement change from fresh reads to low-latency reads during a celebrity event.

Lesson pack: [Week 2 — Storage, indexes, caches, and probabilistic sketches](lessons/week-02.md).

### Week 3: Distribution, replication, and consistency

Read: [Sharding](../reference/data/sharding.md), [Consistent Hashing](../reference/data/consistent-hashing.md), [Range Partitioning](../reference/data/range-partitioning-geohash.md), [Leader-Follower Replication](../reference/data/leader-follower-replication.md), [Quorum](../reference/block/quorum.md), [Replicated Log](../reference/data/replicated-log.md).

Case: design a multi-region order and inventory system. Compare point-lookup sharding with range partitioning, synchronous versus asynchronous replication, and quorum choices. Explicitly handle a hot product and a region partition.

Interview drill: “Design a globally distributed key-value store.” State the consistency contract before discussing technology.

Lesson pack: [Week 3 — Sharding, replication, and consistency](lessons/week-03.md).

### Week 4: Coordination and time

Read: [Raft](../reference/coord/raft.md), [Paxos](../reference/coord/paxos.md), [Gossip](../reference/coord/gossip.md), [Lamport Clock](../reference/block/lamport-clock.md), [Vector Clock](../reference/block/vector-clock.md), [HLC](../reference/block/hybrid-logical-clock.md), [Lease](../reference/block/lease.md), [Heartbeat](../reference/block/heartbeat.md), [Generation Clock](../reference/block/generation-clock.md).

Case: design leader election and membership for a payment-processing cluster. Walk through split brain, delayed messages, clock skew, stale leaders, and fencing. Learn why a lease is not a lock unless stale holders are prevented from acting.

Interview drill: “Design a distributed scheduler.” Explain what must be linearizable and what can be eventually consistent.

## Phase II — Reliable service architecture

### Week 5: Messaging, delivery, and backpressure

Read: [Pub-Sub](../reference/comm/pub-sub.md), [Idempotent Consumer](../reference/comm/idempotent-consumer.md), [Request Batching](../reference/comm/batching-pipelining.md), [Backpressure](../reference/res/backpressure.md), [Throttling](../reference/res/throttling.md), [Claim Check](../reference/arch/claim-check-static-hosting.md).

Case: design a video-upload and processing pipeline. Handle large payloads, duplicate deliveries, slow consumers, poison messages, retries, ordering, and replay.

Interview drill: “Design a notification system” with email, SMS, and push providers that fail independently.

### Week 6: Distributed writes and workflows

Read: [Transactional Outbox](../reference/data/outbox.md), [CDC](../reference/data/cdc.md), [Saga](../reference/data/saga.md), [CQRS](../reference/data/cqrs.md), [Event Sourcing](../reference/data/event-sourcing.md), [Database per Service](../reference/data/database-per-service.md), [2PC](../reference/coord/2pc.md).

Case: design checkout for a marketplace: reserve inventory, authorize payment, create shipment, and send confirmation. Compare 2PC with saga orchestration and choreography. Use outbox + CDC + idempotent consumers to close the dual-write gap.

Interview drill: “Design a payment workflow.” Be precise about retries, compensating actions, reconciliation, auditability, and exactly-once claims.

### Week 7: Resilience and failure containment

Read: [Circuit Breaker](../reference/res/circuit-breaker.md), [Bulkhead](../reference/res/bulkhead.md), [Thundering Herd](../reference/res/thundering-herd.md), [Static Stability](../reference/res/static-stability.md), [Chaos Engineering](../reference/ops/chaos-engineering.md), [Blue-Green/Canary](../reference/ops/blue-green-canary.md), [Feature Flags](../reference/ops/feature-flag.md).

Case: design a ride-hailing dispatch system during a regional outage. Keep critical flows alive, shed optional work, isolate tenants, prevent retry storms, and roll out a fix safely.

Interview drill: “Your dependency is timing out and traffic is rising. What happens next?” Draw the cascading-failure loop and break it at multiple points.

### Week 8: Observability, security, and migration

Read: [Distributed Tracing](../reference/ops/distributed-tracing.md), [Logs and Metrics](../reference/ops/log-aggregation-metrics.md), [Audit Log](../reference/ops/audit-log.md), [API Gateway](../reference/comm/api-gateway.md), [Gatekeeper](../reference/sec/gatekeeper-valet-key.md), [Federated Identity](../reference/sec/federated-identity.md), [Strangler Fig](../reference/arch/strangler-fig.md), [Anti-Corruption Layer](../reference/arch/anti-corruption-layer.md).

Case: migrate a bank’s legacy account platform while preserving auditability and customer access. Design correlation IDs, audit events, progressive rollout, rollback, identity boundaries, and coexistence between old and new models.

Interview drill: “Modernize a monolith without a big-bang rewrite.” Explain how you know the migration is safe.

## Phase III — LLM application foundations

### Week 9: Augmented LLMs and reliable interfaces

Read: [Augmented LLM](../reference-llm/fnd/augmented-llm.md), [Prompting Fundamentals](../reference-llm/fnd/prompting-fundamentals.md), [Structured Outputs](../reference-llm/fnd/structured-outputs.md), [ReAct](../reference-llm/fnd/react.md), [Reasoning Prompts](../reference-llm/fnd/reasoning-prompts.md).

Case: build an insurance-claims intake assistant. Separate extraction from decisions, enforce schemas, validate tool arguments, and define the human-review boundary. Treat model output as an untrusted dependency.

Interview drill: “Design an LLM support assistant.” Specify latency, cost, refusal behavior, escalation, and correctness metrics.

### Week 10: Retrieval and knowledge freshness

Read: [Basic RAG](../reference-llm/ret/basic-rag.md), [Advanced RAG](../reference-llm/ret/advanced-rag.md), [Contextual Retrieval](../reference-llm/ret/contextual-retrieval.md), [Reranking](../reference-llm/ret/reranking.md), [GraphRAG](../reference-llm/ret/graph-rag.md), [Long Context vs RAG](../reference-llm/mem/long-context-vs-rag.md).

Case: build an internal policy assistant for a global company. Design ingestion, chunking, metadata filters, hybrid retrieval, reranking, citations, permissions, freshness, and “I don’t know” behavior. Compare RAG, long context, and a materialized knowledge graph. Add near-duplicate detection with MinHash or SimHash before indexing, and explain why semantic embeddings are not always the right deduplication tool.

Interview drill: “Why did retrieval quality fall after adding more documents?” Diagnose the full pipeline, not only the embedding model.

### Week 11: Context engineering and memory

Read: [Context Engineering](../reference-llm/ctx/context-engineering-overview.md), [Context Rot](../reference-llm/ctx/context-rot.md), [Compaction](../reference-llm/ctx/compaction.md), [Just-in-Time Context](../reference-llm/ctx/just-in-time-context.md), [Progressive Disclosure](../reference-llm/ctx/progressive-disclosure.md), [Structured Note-Taking](../reference-llm/ctx/structured-note-taking.md), [Memory Architectures](../reference-llm/mem/memory-architectures.md), [Prompt Caching](../reference-llm/mem/prompt-caching.md).

Case: design a persistent enterprise research assistant that works across weeks of investigations. Define working, episodic, semantic, procedural, and archival memory; decide what is loaded, summarized, retrieved, or written back; and protect against stale or poisoned memory.

Interview drill: “The agent gets worse as the conversation grows.” Explain attention budget, context rot, compaction, and selective retrieval.

### Week 12: Workflows versus agents

Read: [Workflows vs Agents](../reference-llm/agt/workflows-vs-agents.md), [Prompt Chaining](../reference-llm/wf/prompt-chaining.md), [Routing](../reference-llm/wf/routing.md), [Parallelization](../reference-llm/wf/parallelization.md), [Orchestrator-Workers](../reference-llm/wf/orchestrator-workers.md), [Evaluator-Optimizer](../reference-llm/wf/evaluator-optimizer.md), [Single Agent with Tools](../reference-llm/agt/single-agent-with-tools.md).

Case: build a travel-support automation system. Start with deterministic routing and fixed workflows; add agentic behavior only where the task space is open-ended. Analyze latency, cost, debuggability, failure recovery, and authorization.

Interview drill: “When would you use a workflow instead of a multi-agent system?” Give a decision rule and a concrete boundary.

## Phase IV — Production agents

### Week 13: Agent architecture and orchestration

Read: [Agent Loop](../reference-llm/agt/agent-loop.md), [Sub-Agent Architectures](../reference-llm/agt/sub-agent-architectures.md), [Multi-Agent Orchestration](../reference-llm/agt/multi-agent-orchestration.md), [Maker-Checker](../reference-llm/agt/maker-checker.md), [Generative Agents](../reference-llm/agt/generative-agents.md), [MCP](../reference-llm/proto/mcp.md), [A2A](../reference-llm/proto/a2a.md).

Case: design a software-engineering agent that investigates incidents, proposes a fix, runs tests, and opens a change for approval. Separate planner from executor, isolate context, constrain tools, preserve evidence, and define approval gates.

Interview drill: “Design a coding agent for a large monorepo.” Discuss repository navigation, tool interface design, parallel investigation, checkpoints, and rollback.

### Week 14: Agent engineering, evaluation, and observability

Read: [12-Factor Agents](../reference-llm/eng/12-factor-agents.md), [Harness Design](../reference-llm/eng/harness-design.md), [Brain vs Hands](../reference-llm/eng/brain-vs-hands.md), [LLM Observability](../reference-llm/eng/llm-observability.md), [Eval-Driven Development](../reference-llm/qua/eval-driven-development.md), [Grader Taxonomy](../reference-llm/qua/grader-taxonomy.md), [Capability vs Regression Evals](../reference-llm/qua/capability-vs-regression-evals.md), [End-State Evaluation](../reference-llm/qua/end-state-evaluation.md), [pass@k vs pass^k](../reference-llm/qua/pass-at-k-vs-pass-power-k.md).

Case: launch an agent that resolves customer tickets. Build a trace schema, offline dataset, deterministic checks, model graders, human review, regression suite, and cost/latency dashboards. Distinguish “can do it once” from “reliably does it every time.”

Interview drill: “How do you evaluate an agent?” Answer with task success, safety, process quality, regression protection, and production feedback loops.

### Week 15: Security and containment

Read: [Containment and Blast Radius](../reference-llm/sec/containment-blast-radius.md), [Lethal Trifecta](../reference-llm/sec/lethal-trifecta.md), [Egress Allowlisting](../reference-llm/sec/egress-allowlisting.md), [Browser as Sandbox](../reference-llm/sec/browser-as-sandbox.md), [Permission Boundaries](../reference-llm/sec/auto-mode-permission-boundaries.md), plus [Gatekeeper](../reference/sec/gatekeeper-valet-key.md).

Case: design an accounts-payable agent that reads invoices, accesses vendor systems, and prepares payments. Threat-model prompt injection, data exfiltration, confused deputy behavior, excessive permissions, and irreversible actions. Add least privilege, scoped credentials, egress controls, sandboxing, human approval, and audit trails.

Interview drill: “What is the blast radius if the model is compromised?” Start from assets and capabilities, then show containment layers.

### Week 16: Model choice, rollout, and capstone synthesis

Read: [Fine-Tuning vs Prompting](../reference-llm/model/fine-tuning-vs-prompting.md), [Preference Tuning](../reference-llm/model/preference-tuning.md), [Reasoning Models](../reference-llm/model/reasoning-models.md), [Speculative Decoding](../reference-llm/model/speculative-decoding.md), [Rainbow Deployments](../reference-llm/eng/rainbow-deployments.md).

Capstone: design a healthcare operations copilot that retrieves policy and patient-context data, coordinates deterministic tools, drafts recommendations, escalates high-risk cases, and learns from feedback. Submit a system diagram, data contracts, threat model, eval plan, SLOs, cost model, rollout plan, and two requirement-change adaptations.

Capstone change requests:

- traffic grows 20× and the model provider has a partial outage;
- regulations require regional data isolation and complete decision provenance;
- users report plausible but unsupported answers;
- a new model improves quality but doubles latency and cost.

## Pattern families to master

These are the repository’s highest-value transfer structures:

| Problem shape | Patterns to connect | Core intuition |
|---|---|---|
| Keep state and events consistent | Outbox → CDC → Idempotent Consumer | Make one durable commit authoritative; make delivery repeatable. |
| Make reads cheap | Cache → Materialized View → Index → CQRS | Spend write/stream work to avoid repeated read work. |
| Coordinate without shared time | Logical clocks → leases/heartbeats → quorum → Raft | Replace unreliable clocks with explicit ordering, ownership, and overlap. |
| Survive dependency failure | Bulkhead → Circuit Breaker → Backpressure → Static Stability | Isolate, stop, signal, and pre-provision before failure cascades. |
| Aggregate across boundaries | API Composition → CQRS read model → GraphQL Federation | Choose runtime composition versus precomputed composition based on freshness and coupling. |
| Control model behavior | Structured output → tools → workflow → agent loop | Increase autonomy only as far as the task’s uncertainty requires. |
| Fit knowledge into attention | JIT context → progressive disclosure → retrieval → compaction/memory | Context is a managed resource with an attention budget. |
| Make agents trustworthy | Harness → traces → evals → rollout → containment | Reliability is a system property, not a prompt property. |

## Interview practice protocol

For every problem, use this sequence:

1. Clarify users, APIs, scale, latency, durability, consistency, privacy, and failure tolerance.
2. State the simplest viable architecture and the invariant it protects.
3. Draw the critical read/write path.
4. Deep-dive on one bottleneck and one failure mode.
5. Name the operational signals, tests, and rollout strategy.
6. Explain the rejected alternative and the threshold that would change your choice.
7. Close with a requirement change and adapt the design.

Score yourself from 0–3 on each dimension: requirements, decomposition, data model, scale, consistency, failure handling, observability, security, trade-offs, and communication. A strong interview answer is not the one with the most patterns; it is the one whose patterns are consequences of explicit requirements.

## Suggested sequence of case studies

Use these as spaced-review prompts after the main weeks: URL shortener, news feed, ride dispatch, chat, notification platform, payment workflow, search engine, analytics pipeline, feature-flag service, document assistant, support agent, coding agent, research agent, and high-risk financial agent.

Each repeat should add one constraint—multi-region operation, adversarial traffic, strict auditability, tenfold scale, provider outage, or a tighter cost budget. The pattern should become recognizable when the constraint changes, not memorized as a fixed diagram.

## Definition of mastery

You have mastered a pattern when you can:

- explain the problem without naming the pattern;
- derive the pattern from the invariant and forces;
- draw its normal, failure, and recovery paths from memory;
- compare it with at least two alternatives;
- identify its operational and security failure modes;
- combine it with neighboring patterns without creating contradictory guarantees;
- adapt it under a new scale, consistency, cost, or safety constraint;
- explain it clearly in five minutes and defend it for twenty.
