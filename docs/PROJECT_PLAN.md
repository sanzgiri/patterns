# System Design and LLM Patterns Teaching Plan

## Objective

Build a self-paced, case-study-driven tutorial that develops deep intuition for system-design and LLM/agentic patterns, with repeated practice applying them to real company problems and technical interviews.

## Completed

### Repository analysis

- Reviewed the system-design reference corpus: 85 pattern pages.
- Reviewed the LLM/agentic reference corpus: 60 pattern pages.
- Identified the major pattern families and their relationships.
- Clarified the role of Bloom filters, HyperLogLog, Count-Min Sketch, and MinHash.

### Curriculum design

- Created a complete 21-week curriculum map in [curriculum.md](curriculum.md).
- Organized the curriculum into:
  - Weeks 1–4: distributed-systems foundations
  - Weeks 5–8: reliable service architecture
  - Weeks 9–12: LLM application foundations
  - Weeks 13–16: production agents
  - Weeks 17–20: system-design breadth pass
  - Week 21: LLM/agent breadth extensions
- Assigned every system-design reference page to at least one week.
- Assigned every LLM/agent reference page to at least one week.
- Added company-style case studies, interview drills, deliverables, and mastery criteria.
- Added a distinction between breadth exposure and actual mastery.

### Detailed lesson material

Created initial lesson packs in `docs/lessons/`:

- [Week 1 — Requirements, boundaries, and scaling shape](lessons/week-01.md)
- [Week 2 — Storage, indexes, caches, and probabilistic sketches](lessons/week-02.md)
- [Week 3 — Sharding, replication, and consistency](lessons/week-03.md)

Each lesson includes:

- learning outcomes;
- required patterns and readings;
- concept explanations;
- worked company case;
- requirement changes and failure analysis;
- practice questions;
- interview drill;
- self-assessment rubric.

### Web tutorial

- Added a Guided Tutorial section to the MkDocs site.
- Added tutorial navigation and a landing page.
- Added web versions of Weeks 1–3 in `site_src/tutorial/`.
- Structured each web lesson as:
  1. Lesson
  2. Worked example
  3. Practice
  4. Interview drill
- Repaired the stale local Python virtual environment.
- Installed the MkDocs documentation dependencies.
- Verified that `mkdocs build --strict` succeeds.
- Validated tutorial links with no missing relative links.

## Remaining work

### High priority: complete the web tutorial

Write web-based lesson pages for Weeks 4–21. Each should follow the same structure as Weeks 1–3.

#### Weeks 4–8 — Reliable distributed services

- Week 4: coordination, logical time, leases, heartbeats, and leader election
- Week 5: messaging, delivery semantics, idempotency, batching, and backpressure
- Week 6: outbox, CDC, saga, CQRS, event sourcing, and distributed writes
- Week 7: circuit breakers, bulkheads, throttling, static stability, chaos, and progressive delivery
- Week 8: tracing, metrics, logs, identity, auditability, and strangler migrations

#### Weeks 9–12 — LLM application foundations

- Week 9: augmented LLMs, prompting, structured outputs, tools, and ReAct
- Week 10: RAG, hybrid retrieval, reranking, GraphRAG, citations, and freshness
- Week 11: context engineering, context rot, compaction, progressive disclosure, and memory
- Week 12: workflows versus agents, routing, chaining, parallelization, and evaluators

#### Weeks 13–16 — Production agents

- Week 13: agent loops, tool use, sub-agents, multi-agent orchestration, MCP, A2A, and maker-checker
- Week 14: harness design, observability, eval-driven development, graders, regression tests, and reliability metrics
- Week 15: containment, prompt injection, egress controls, browser sandboxes, and permission boundaries
- Week 16: model selection, reasoning models, fine-tuning, preference tuning, speculative decoding, rollout, and capstone synthesis

#### Weeks 17–20 — System-design breadth pass

- Week 17: service templates, chassis, sidecars, meshes, registries, discovery, ambassadors, state watch, and configuration
- Week 18: API Composition, BFF, GraphQL Federation, contract tests, and component tests
- Week 19: segmented logs, CRDTs, Merkle trees, high/low watermarks, clock-bound waits, singular update queues, consistent cores, and emergent leaders
- Week 20: cache-managed strategies, polyglot persistence, space-based architecture, edge computing, compute consolidation, and enterprise identity

#### Week 21 — LLM/agent extensions

- Coding agents
- Computer use
- Self-directed swarms
- Context plumbing and reinforcement
- Eval saturation
- AGENTS.md
- Code Mode
- SKILL.md / agent skills

### Medium priority: improve learning interaction

- Add answer reveal sections or collapsible worked solutions.
- Add progress checkboxes and a lightweight completion tracker.
- Add diagrams directly to tutorial lessons where they materially improve understanding.
- Add spaced-review links from later lessons back to earlier patterns.
- Add a consistent interview scoring sheet for every case study.
- Add a capstone submission template.

### Medium priority: reduce content duplication

The Markdown lesson packs in `docs/lessons/` and the web pages in `site_src/tutorial/` currently contain overlapping content. Choose one canonical source and generate or include the other where practical.

### Lower priority: publish the tutorial

- Decide whether to publish through the existing GitHub Pages workflow.
- Confirm the Pages source is configured for GitHub Actions.
- Push the completed tutorial content to the main branch.
- Verify the public tutorial URL and navigation after deployment.

## Current verification status

- Curriculum links: validated.
- Tutorial links: validated.
- MkDocs strict build: passes.
- Local server: not currently running.

To run locally:

```bash
.venv/bin/mkdocs serve --dev-addr 127.0.0.1:8000
```

Open:

```text
http://127.0.0.1:8000/patterns/tutorial/
```

## Definition of done

The project is complete when:

- all 21 weeks have web-based teaching material;
- every lesson teaches before testing;
- every pattern has an explicit lesson location;
- every week includes a realistic company case;
- every week includes practice and an interview drill;
- all links and the strict MkDocs build pass;
- the tutorial is published or has a documented local run path;
- the learner can submit answers and receive targeted feedback through the teaching workflow.

